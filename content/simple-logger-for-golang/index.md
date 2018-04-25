+++
date = "2016-01-18T00:47:11+05:00"
draft = false
title = "go-logger – простой и информативный логгер для Go"
type = "post"
aliases = [
	"post/simple-logger-for-golang"
]
tags = [ "golang", "go", "log", "logger", "cli" ]

+++
Не смотря на всё многообразие существующих логгеров для **Go**, как-то не удалось подобрать удобный и подходящий для меня. Хотелось иметь инструмент, похожий на те, какими пользовался в других языках. Если хочешь сделать что-то хорошо, сделай это сам.

<!--more-->

**Update 3 августа 2016 года.**

> Здесь имеется ввиду логирование какой-то информации в консоль, либо файл. Случай со структурированными логами рассматривается в конце статьи.

Итак, мой [go-logger](https://github.com/ivahaev/go-logger). Что он умеет и чем он лучше других:

* Все методы логгирования принимают любое количество аргументов любых типов;
* Метод [Debug()](https://godoc.org/github.com/ivahaev/go-logger#Debug) красиво выводит данные. Если это структура, слайс или мапа, то будет наглядно видно что в них, если же это простая переменная, то увидим тип и размер;
* Отображается файл и строка в которой вызвано логгирование;
* Цветовое кодирование каждого уровня;
* Возможность изменить формат вывода даты.

#### Пример

```go
    package main

    import "github.com/ivahaev/go-logger"

    func main() {
        logger.SetLevel("DEBUG")

        logger.Debug("Some string for debug", 123, map[string]interface{}{"prop1": "val1", "prop2": 321})
        logger.Info("Some string for info", 123, map[string]interface{}{"prop1": "val1", "prop2": 321})
        logger.Notice("Some string for debug", 123, map[string]interface{}{"prop1": "val1", "prop2": 321})
        logger.Warn("Some string for warning", 123, map[string]interface{}{"prop1": "val1", "prop2": 321})
        logger.Error("Some string for error", 123, map[string]interface{}{"prop1": "val1", "prop2": 321})
        logger.Crit("Some string for critical", 123, map[string]interface{}{"prop1": "val1", "prop2": 321})
    }
```

Вот что получим в консоли:
{{< img src="logger-console.png" alt="logger console output" width="1024" >}}

Со временем добавлю возможность прятать файл вызова и строку, такие мысли уже посещали для продакшена. Может быть, что-то ещё, но пока этот логгер меня устраивает на 100%, я его использую во всех своих проектах. Возможно, пригодится и вам!

#### P.S. 3 августа 2016 года

В данный момент полностью перешёл на [logrus](https://github.com/Sirupsen/logrus), т.к. стал использовать аггрегирование логов в [Graylog2](https://www.graylog.org/), и он здесь удобнее, я оценил всю прелесть структурированных логов. Свой же логгер использую для дебага в процессе разработки, в основном, метод [Debug()](https://godoc.org/github.com/ivahaev/go-logger#Debug).

#### P.P.S. 25 апреля 2018 года

Любовь к [logrus](https://github.com/Sirupsen/logrus) прошла, чему сильно способствовало переименование аккаутна разработчика на Гитхабе с `Sirupsen` на `sirupsen`, что сломало сборку нескольких моих проектов. Давно руки чесались попробовать [zap](https://github.com/uber-go/zap), и создатель Логруса сам того не ведая, меня стимулировал к переходу. Работает шустро, возможностей достаточно для моих задач, теперь дружу с ним :).