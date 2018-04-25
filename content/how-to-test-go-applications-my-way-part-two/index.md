+++
date = "2018-04-21T11:55:00+05:00"
title = "Мой подход к тестированию. Часть вторая"
slug = "how-to-test-go-applications-my-way-part-two"
draft = false
type = "post"

tags = [ "go", "golang", "test", "testing", "goconvey", "goblin" ]
+++

В [первой статье](/post/how-to-test-go-applications-my-way-part-one/) цикла о тестировании, которая вышла почти два года назад, я описал свой подход к тестированию, который был актуален в 2016 году. Время идёт, всё изменяется, стандартная библиотека **Go** становится лучше и вот пару-тройку мажорных версий назад в пакете [testing](https://golang.org/pkg/testing/) появился новый метод [Run()](https://golang.org/pkg/testing/#T.Run), который позволяет запускать именованные подтесты. Теперь [Гоблина](https://github.com/franela/goblin) можно отправить обратно в пещеру и уменьшить число зависимостей проекта.

<!--more-->

### Run tester Run

Не будем вспоминать, как мы без этого жили раньше, сразу же рассмотрим пример. Имеется у нас функция странного умножения:

```go
func Multiply(a, b int) int {
    res := a * b
    if res == 0 {
        return 0
    }

    if res % 2 == 0 {
        return res - 1
    }

    return res + 1
}
```

Нам нужно её протестировать. Как обычно, используем табличное тестирование, но теперь мы можем вложить дополнительную информацию в таблицу и вывести её:

```go
package multi

import (
    "fmt"
    "testing"
)

func TestMultiply(t *testing.T) {
    testData := []struct{
        name string
        a, b, res int
    }{
        {
            name: "First",
            a: 1,
            b: 1,
            res: 2,
        },
        {
            name: "Second",
            a: 2,
            b: 1,
            res: 1,
        },
        {
            name: "First",
            a: 2,
            b: 2,
            res: 3,
        },
    }

    for i, v := range testData {
        t.Run(fmt.Sprintf("#%d name: %s", i, v.name), func(t  *testing.T) {
            res := Multiply(v.a, v.b)
            if res != v.res {
                t.Errorf("Multiply(%d, %d) => %d, expected %d", v.a, v.b, res, v.res)
            }
        })
    }
}
```

Выполнив данный тест, мы увидим следующее:

```bash
go test -v
=== RUN   TestMultiply
=== RUN   TestMultiply/#0_name:_First
=== RUN   TestMultiply/#1_name:_Second
=== RUN   TestMultiply/#2_name:_First
--- PASS: TestMultiply (0.00s)
    --- PASS: TestMultiply/#0_name:_First (0.00s)
    --- PASS: TestMultiply/#1_name:_Second (0.00s)
    --- PASS: TestMultiply/#2_name:_First (0.00s)
PASS
ok  	multi	0.008s
```

Теперь усложним нашу функцию. Предположим, мы хотим к результату кратному десяти прибавлять пять. Добавим этот случай в таблицу:

```go
    testData := []struct{
        name string
        a, b, res int
    }{
        {
            name: "First",
            a: 1,
            b: 1,
            res: 2,
        },
        {
            name: "Second",
            a: 2,
            b: 1,
            res: 1,
        },
        {
            name: "First",
            a: 2,
            b: 2,
            res: 3,
        },
        {
            name: "Twenty",
            a: 2,
            b: 10,
            res: 25,
        },
    }
```

Тут же видим на каком элементе наш тест завалился:

```bash
go test -v
=== RUN   TestMultiply
=== RUN   TestMultiply/#0_name:_First
=== RUN   TestMultiply/#1_name:_Second
=== RUN   TestMultiply/#2_name:_First
=== RUN   TestMultiply/#3_name:_Twenty
--- FAIL: TestMultiply (0.00s)
    --- PASS: TestMultiply/#0_name:_First (0.00s)
    --- PASS: TestMultiply/#1_name:_Second (0.00s)
    --- PASS: TestMultiply/#2_name:_First (0.00s)
    --- FAIL: TestMultiply/#3_name:_Twenty (0.00s)
    	ttt_test.go:43: Multiply(2, 10) => 19, expected 25
FAIL
exit status 1
FAIL     _/multi    0.008s
```

Теперь подправим нашу функцию:

```go
func Multiply(a, b int) int {
    res := a * b
    if res == 0 {
        return 0
    }

    if res % 10 == 0 {
        return res + 5
    }

    if res % 2 == 0 {
        return res - 1
    }

    return res + 1
}
```

Всё встало на свои места, все проверки пройдены:

```bash
go test -v
=== RUN   TestMultiply
=== RUN   TestMultiply/#0_name:_First
=== RUN   TestMultiply/#1_name:_Second
=== RUN   TestMultiply/#2_name:_First
=== RUN   TestMultiply/#3_name:_Twenty
--- PASS: TestMultiply (0.00s)
    --- PASS: TestMultiply/#0_name:_First (0.00s)
    --- PASS: TestMultiply/#1_name:_Second (0.00s)
    --- PASS: TestMultiply/#2_name:_First (0.00s)
    --- PASS: TestMultiply/#3_name:_Twenty (0.00s)
PASS
ok  	multi	0.008s
```

### Покрытие

Не смотря на то, что проверки в тесте выполнены успешно, мы не можем с уверенностью сказать, что протестировали все возможные результаты. Значительно повысить (либо наоборот) уверенность нам поможет ключ `-cover` команды `go test`. Запустим выполнение тестов нашего примера с ключом `-cover`:

```bash
go test -cover
PASS
coverage: 87.5% of statements
ok  	multi	0.009s
```

Да, опасения были небеспочвенны, у нас покрыто только 87.5%. Наш старый знакомый [GoConvey](http://goconvey.co/) в очень удобном виде покажет какие именно части кода не покрыты тестами:
{{< img src="multi-cover-01.png" alt="test coverage 01" width="540" >}}

У нас не отработан кейс, при котором произведение получается равным нулю, следует его добавить:
```go
    testData := []struct{
        name string
        a, b, res int
    }{
        {
            name: "First",
            a: 1,
            b: 1,
            res: 2,
        },
        {
            name: "Second",
            a: 2,
            b: 1,
            res: 1,
        },
        {
            name: "First",
            a: 2,
            b: 2,
            res: 3,
        },
        {
            name: "Twenty",
            a: 2,
            b: 10,
            res: 25,
        },
        {
            name: "Zero",
            a: 2,
            b: 0,
            res: 0,
        },
    }
```

Проверяем покрытие:

```bash
go test -cover
PASS
coverage: 100.0% of statements
ok  	multi	0.007s
```

С таким покрытием не стыдно и в продакшн!