+++
date = "2016-01-14T21:52:24+05:00"
draft = false
title = "Amigo – лучший друг Golang и Asterisk"
slug = "amigo-best-frend-of-golang-and-asterisk"
banner = "/img/three-amigos.jpg"
tags = [ "asterisk", "golang", "go", "ami", "amigo" ]
aliases = [
	"amigo-best-frend-of-golang-and-asterisk"
]
+++
Начиная писать свой первый [Peach Dialer](http://peach-dialer.com), я выбрал язык, который довольно хорошо знал, благо работал с ним с 1999 года&nbsp;&mdash; **PHP**. Меня не смущало, что он, в принципе, не предназначен для создания долгоживущих процессов, но то, что получилось в итоге, работает по несколько месяцев и радует своих владельцев.

Потом пошла мода на [Websocket](https://ru.wikipedia.org/wiki/WebSocket), который захотелось применить в интерфейсе, а **PHP** в то время не очень-то умел с ними работать (может, и сейчас не умеет). Я обратил внимание на [node.js](https://nodejs.org). Классная штука, любовь на века, подумал я, но вскоре захотелось большего.

Бо&#769;льшим для меня стал [Go](https://golang.org)&nbsp;&mdash; язык, совмещающий скорость и удобство деплоя компилируемых языков с простотой разработки, свойственной интерпретируемым языкам. К сожалению, разработанных библиотек надлежащего качества и с ожидаемым функционалом для взаимодействия с **Asterisk** в то время не было. Были какие-то начинания, но не доведённые до конца.

Итак, по сусекам поскребя, да по амбару пометя&#769;, испёк я [Amigo](https://github.com/ivahaev/amigo)&nbsp;&mdash; удобную библиотеку для работы с астериском посредством AMI протокола.

<!--more-->

Установка библиотеки:

```go
go get github.com/ivahaev/amigo
```

Пример использования (здесь применяется ещё мой простой [логгер](https://github.com/ivahaev/go-logger)):

```go
import (
    "github.com/ivahaev/amigo"
    "github.com/ivahaev/go-logger"
)

// Создаем функции-обработчики событий
// amigo.M – это псевдоним для map[string]string
func DeviceStateChangeHandler (m amigo.M) {
    logger.Debug(`Принято событие "DeviceStateChange"`, m)
}

func DefaultHandler (m amigo.M) {
    logger.Debug("Принято событие", m)
}


func main() {

    // Подключаемся к астериску.
    // Имя пользователя и пароль являются обязательными параметрами.
    // По умолчанию хост – "127.0.0.1", порт – "5038".
    a := amigo.New("username", "password", "host", "port")
    a.Connect()


    //Регистрируем обработчик для события "DeviceStateChange"
    a.RegisterHandler("DeviceStateChange", DeviceStateChangeHandler)

    // Регистрируем обработчик для всех событий
    a.RegisterDefaultHandler(DefaultHandler)

    // Также можем создать канал для приема всех событий
    // и зарегистрировать его
    c := make(chan map[string]string, 100)
    a.SetEventChannel(c)


    // Проверим, что подключение успешно и отправим запрос "QueueSummary"
    if a.Connected() {
        result, err := a.Action(amigo.M{"Action": "QueueSummary", "ActionID": "Init"})
        // Проверяем на ошибку и обрабатываем результат. Ответ на запрос придёт в соответствующем событии.
        // Ловить его нужно в общем канале, обработчике по умолчанию, либо в специальном обработчике.
    }
}
```

Стоит заметить, что подключение будет автоматически восстанавливаться при обрыве связи.

В следующий раз расскажу как использовать библиотеку в режиме **ASYNC:AGI**.