+++
date = "2016-05-01T14:12:32+05:00"
draft = false
title = "Прокачанный таймер на Go"
slug = "advanced-go-timer"
banner = "/img/sandClock.png"
tags = ["go", "golang", "timer", "time", "pause"]
aliases = [
	"advanced-go-timer"
]
+++
У языка [Go](https://golang.org/) шикарная стандартная библиотека, инструменты на все случаи жизни, при этом достаточно лаконичные. Например, рассмотрим отличный пакет [time](https://golang.org/pkg/time/). За всё время работы с **Go**, мне всего лишь дважды приходилось расширять его возможности.

Первый раз, около года назад, понадобилось строковое представление времени и периодов на русском языке, что привело к созданию пакета [russian-time](https://github.com/ivahaev/russian-time). Он не очень красивый, создан на скорую руку, потому не буду на нём останавливаться сегодня.

Второй инструмент, мне кажется более интересным. Предпосылкой к созданию этого небольшого пакета, была необходимость контроля времени выполнения скриптов во встроенном интерпретаторе **JavaScript** **Google V8**. Так уж повелось, что **JavaScript**, как правило, характеризуется асинхронным поведением, что несколько затрудняло выполнение поставленной задачи. Одним из компонентов решения должен был стать таймер, который можно приостанавливать на неопределённое время, а после запускать снова с момента остановки. Так появился [timer](https://github.com/ivahaev/timer).

<!--more-->

Функции [AfterFunc](https://godoc.org/github.com/ivahaev/timer#AfterFunc)  и [NewTimer](https://godoc.org/github.com/ivahaev/timer#NewTimer)  очень похожи на стандартные, но у возвращаемого указателя на структуру [Timer](https://godoc.org/github.com/ivahaev/timer#Timer) есть два новых метода:

*  [Pause](https://godoc.org/github.com/ivahaev/timer#Timer.Pause)
*  [Start](https://godoc.org/github.com/ivahaev/timer#Timer.Start)

**Pause** приостанавливает выполнение таймера, а **Start**, соответственно, запускает его с момента остановки, либо после инициализации. Т.е. после создания таймера, запускать его требуется вручную.

Пример из теста лучше покажет идею:

```go
var now = time.Now()

// Создаем таймер на 1 секунду
var timer = AfterFunc(time.Second, func() {

    // Сравниваем сколько прошло времени
	if time.Now().Sub(now) < time.Second*2 {
		panic("Early func")
	}
})

// Запускаем таймер
timer.Start()

// Через пол секунды приостанавливаем
time.Sleep(time.Microsecond * 500)
timer.Pause()

// Через секунду запускаем снова
time.Sleep(time.Second)
timer.Start()
time.Sleep(time.Second)
```

[![GoDoc](https://godoc.org/github.com/ivahaev/timer?status.svg)](https://godoc.org/github.com/ivahaev/timer)