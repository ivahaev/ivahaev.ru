+++
date = "2015-11-11T01:53:02+05:00"
draft = false
title = "Проигрывание файла при телефонном разговоре"
slug = "proighryvaniie-faila-pri-tieliefonnom-razghovorie"
tags = [ "asterisk", "callcenter", "freepbx", "telephony" ]
type = "post"
aliases = [
	"post/proighryvaniie-faila-pri-tieliefonnom-razghovorie"
]
+++
В работе оператора колл-центра часто возникает необходимость проиграть в канал заранее заготовленные голосовые файлы, например, адреса офисов и режимы работы.

Рассмотрим вариант решения данной задачи на базе **IP PBX Asterisk**.

<!--more-->

Пример для **FreePBX**, для голого астериска все делается ещё проще.

Первым делом, нужно добавить сочетание клавиш на вызов макроса в `features.conf`, или для **FreePBX** в `features_applicationmap_custom.conf`:

```
playsound => *6,callee,macro,playsound
```

По нажатию в разговоре ***6** вызовется макрос playsound

Важный момент, нужно добавить возможность вызова макроса. Для этого либо в дайлплане присваиваем переменную:

```
exten => _+x.,n,Set(DYNAMIC_FEATURES=playsound)
```

Либо в **extensions.conf** пишем в секции **[globals]**:
```
DYNAMIC_FEATURES=playsound
```

Макрос и нужный контекст в дайлплане:
```
[macro-playsound]

exten => s,1,AGI(create_spy.php,${CHANNEL})

; end of [macro-playsound]

[spy]
exten => 1,1,Chanspy(${chan},v(-4)wqBbE)
exten => 1,n,Hangup

exten => 2,1,Answer
exten => 2,n,Set(VOLUME(TX)=-3)
exten => 2,n,Playback(${audio})
exten => 2,n,Hangup

; end of [spy]
```

 Файл **create_spy.agi**:

```
#!/usr/bin/php -q

<?php

$par=$argv[1];
$channel= $par;

    $uid=time();
    $f1=fopen("/tmp/$uid.call","w");
    fputs($f1,"Channel: LOCAL/2@spy\n");
    fputs($f1,"MaxRetries: 0\n");
    fputs($f1,"RetryTime: 600\n");
    fputs($f1,"WaitTime: 30\n");
    fputs($f1,"Context: spy\n");
    fputs($f1,"Extension: 1\n");
    fputs($f1,"Priority: 1\n");
    fputs($f1,"Set: audio=custom/VOICE_FILE\nSet: chan=$channel\n");
    fclose($f1);
    system("mv /tmp/$uid.call /var/spool/asterisk/outgoing/");
```
Здесь **VOICE_FILE** - это проигрываемый файл.  Не забыть сделать `chmod +x /var/lib/asterisk/agi-bin/create_spy.php` ну или где в нашем дистрибутиве расположены **agi** скрипты.

Само собой, можно сделать все через **AMI**, в данном случае, была задача показать вариант решения вопроса. Реализация за вами!