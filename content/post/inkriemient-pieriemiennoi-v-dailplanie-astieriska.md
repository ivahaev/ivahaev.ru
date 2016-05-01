+++
date = "2015-11-14T12:34:30+05:00"
draft = false
title = "Инкремент переменной в дайлплане Астериска"
slug = "inkriemient-pieriemiennoi-v-dailplanie-astieriska"
tags = ["asterisk", "dialplan", "counter", "increment", "telephony"]
aliases = [
	"inkriemient-pieriemiennoi-v-dailplanie-astieriska"
]
+++
Если требуется реализовать какой-то счетчик в дайлплане, удобно использовать переменную канала. Но просто так изменить её значение, прибавляя единицу, не получится. <!--more--> Нужно использовать выражение:

```
same => n,Set(REP=$[${REP} + 1])
```