+++
date = "2016-01-18T00:47:11+05:00"
draft = false
title = "go-logger – простой и информативный логгер для Go"
slug = "simple-logger-for-golang"
aliases = [
	"simple-logger-for-golang"
]
tags = [ "golang", "go", "log", "logger", "cli" ]

#disable_profile = true
#disable_widgets = true
+++
Не смотря на всё многообразие существующих логгеров для **Go**, как-то не удалось подобрать удобный и подходящий для меня. Хотелось иметь инструмент, похожий на те, какими пользовался в других языках. Если хочешь сделать что-то хорошо, сделай это сам.

<!--more-->

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
![output](/img/logger-console.png)

Со временем добавлю возможность прятать файл вызова и строку, такие мысли уже посещали для продакшина. Может быть, что-то ещё, но пока этот логгер меня устраивает на 100%, я его использую во всех своих проектах. Возможно, пригодится и вам!