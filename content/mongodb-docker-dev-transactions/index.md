+++
date = "2018-07-10T23:11:00+05:00"
draft = false
title = "ACID Транзакции и MongoDB"
type = "post"

tags = [ "mongodb", "acid", "transactions", "docker", "cluster"]

+++

Летом 2018 года (т.е. прямо сейчас, на момент написания данной статьи) случилось невероятное&nbsp;&mdash; в [MongoDB](https://www.mongodb.com/) завезли честные [ACID транзакции](https://ru.wikipedia.org/wiki/ACID). С выходом четвёртой версии этой документ-ориентированной [СУБД](https://ru.wikipedia.org/wiki/%D0%A1%D0%B8%D1%81%D1%82%D0%B5%D0%BC%D0%B0_%D1%83%D0%BF%D1%80%D0%B0%D0%B2%D0%BB%D0%B5%D0%BD%D0%B8%D1%8F_%D0%B1%D0%B0%D0%B7%D0%B0%D0%BC%D0%B8_%D0%B4%D0%B0%D0%BD%D0%BD%D1%8B%D1%85), её можно использовать для чуть более серьёзных приложений.

Для тех, кто в танке, в двух словах: транзакции позволяют нам провести серию изменений в нескольких документах и сохранить их разом, либо так же разом отменить все вносимые в рамках транзакции изменения, если что-то пошло не так, либо произошел сбой в приложении.

К сожалению, разработчику воспользоваться этой супер-фичей не так-то просто. Ниже я расскажу почему, и что с этим всем делать.

<!--more-->

Если открыть документацию к СУБД на разделе [Транзакции](https://docs.mongodb.com/master/core/transactions/), можем увидеть следующую ремарку:

> Multi-document transactions are available for replica sets only. Transactions for sharded clusters are scheduled for MongoDB 4.2

Это нам говорит о том, что простой сервер MongoDB не поддерживает транзакции, только кластер в режиме **replica set**. Поддержка в **sharded** кластерах также будет позже, в версии 4.2.

При этом, обычный сервер даст нам начать транзакцию, сохранить её, или отменить, но ничего в рамках неё сделать не получится, будет выведена примерно такая ошибка:

```mongodb
WriteCommandError({
  "ok" : 0,
  "errmsg" : "Transaction numbers are only allowed on a replica set member or mongos",
  "code" : 20,
  "codeName" : "IllegalOperation"
})
```

К счастью, каждый может запустить у себя кластер MongoDB, состоящий из одного сервера. На своих машинах, на которых я занимаюсь разработкой, все СУБД я запускаю в [docker](http://docker.com/) контейнерах. Например, запуск обычного сервера MongoDB выглядит так:

```bash
docker run -v ~/mongo/:/data/db --name mongo --restart=always -p 27017:27017 -d mongo mongod --smallfiles
```

Разберём ключи запуска:

- `-v ~/mongo/:/data/db` означает примонтировать локальную директорию `~/mongo/` в `/data/db` контейнера, таким образом, сама база будет храниться на хост-машине, что позволит нам удалять запущенный контейнер, обновлять версии и т.д. с сохраненим наших данных;
- `--name mongo` задаёт имя контейнеру;
- `--restart=always` говорит о том, что при любых падениях сервиса в контейнере, его следует перезапустить, а так же, стартовать контейнер после загрузки операционной системы;
- `-p 27017:27017` &laquo;прокидывает&raquo; порт на хост машину;
- `-d` указывает на то, что нужно запустить контейнер в виде демона;
- `mongo`&nbsp;&mdash; имя образа для запуска контейнера;
- `mongod --smallfiles`&nbsp;&mdash; команда для запуска сервиса в контейнере.

Как запускать простой сервер, я привёл просто для справки. Теперь давайте разбираться, что необходимо сделать для запуска сервера, поддерживающего транзакции.

Первым делом, следует создать новую сеть внутри докера, в которой будут работать все серверы нашего кластера. Да, я писал выше, что сервер будет один, но сеть нужно создать обязательно, иначе ничего не получится.

```bash
docker network create mongo-cluster
```

Далее в параметрах запуска контейнера нужно указать использование новой сети `--net mongo-cluster`, а также передать параметр серверу, для работы в режиме **replica set:** `--replSet rs0`. Также, я намеренно опустил ключ `--restart=always`, т.к. не всегда использую MongoDB в работе в настоящее время и не хочу, чтобы она стартовала вместе с операционной системой.

```bash
docker run -v ~/mongo/:/data/db --name mongo -p 27017:27017 -d mongo mongod --smallfiles --replSet rs0
```

Отлично, контейнер запущен, в чем мы можем убедиться, выполнив команду `docker ps` и увидев примерно следующее:

```bash
docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS                      NAMES
2292d7e0778b        mongo               "docker-entrypoint.s…"   About a minute ago   Up About a minute   0.0.0.0:27017->27017/tcp   mongo
```

Далее нам нужно инициализировать кластер, для этого войдём в консоль запущенного сервера, создадим конфигурацию нашего кластера и проведём инициализацию:

```bash
docker exec -it mongo mongo
# output omited #

> config = {
    "_id" : "rs0",
    "members" : [
        {
            "_id" : 0,
            "host" : "mongo:27017"
        }
    ]
}
> rs.initiate(config)
{
	"ok" : 1,
	"operationTime" : Timestamp(1531248932, 1),
	"$clusterTime" : {
		"clusterTime" : Timestamp(1531248932, 1),
		"signature" : {
			"hash" : BinData(0,"AAAAAAAAAAAAAAAAAAAAAAAAAAA="),
			"keyId" : NumberLong(0)
		}
	}
}
rs0:SECONDARY>
rs0:PRIMARY>
```

Готово! Мы получили кластер из одного сервера MongoDB. Теперь можно проверить, что всё работает, как ожидается.

```bash
rs0:PRIMARY> session = db.getMongo().startSession()
session { "id" : UUID("7eb81006-983f-4398-adc7-5ed23e027377") }
rs0:PRIMARY> database = session.getDatabase("test")
test

rs0:PRIMARY> // Создадим несколько документов
rs0:PRIMARY> database.col.insert({name: "1"})
WriteResult({ "nInserted" : 1 })
rs0:PRIMARY> database.col.insert({name: "2"})
WriteResult({ "nInserted" : 1 })
rs0:PRIMARY> database.col.insert({name: "3"})
WriteResult({ "nInserted" : 1 })
rs0:PRIMARY> database.col.insert({name: "4"})
WriteResult({ "nInserted" : 1 })

rs0:PRIMARY> // Посмотрим, что у нас получилось
rs0:PRIMARY> database.col.find({})
{ "_id" : ObjectId("5b45026edc396f534f11952f"), "name" : "1" }
{ "_id" : ObjectId("5b450272dc396f534f119530"), "name" : "2" }
{ "_id" : ObjectId("5b450274dc396f534f119531"), "name" : "3" }
{ "_id" : ObjectId("5b450276dc396f534f119532"), "name" : "4" }

rs0:PRIMARY> // Начинаем транзакцию
rs0:PRIMARY> session.startTransaction()

rs0:PRIMARY> // Изменим один документ
rs0:PRIMARY> database.col.update({name: "4"}, {name: "44"})
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })

rs0:PRIMARY> // Проверим изменения
rs0:PRIMARY> database.col.find({})
{ "_id" : ObjectId("5b45026edc396f534f11952f"), "name" : "1" }
{ "_id" : ObjectId("5b450272dc396f534f119530"), "name" : "2" }
{ "_id" : ObjectId("5b450274dc396f534f119531"), "name" : "3" }
{ "_id" : ObjectId("5b450276dc396f534f119532"), "name" : "44" }

rs0:PRIMARY> // Можно открыть соседний терминал и убедиться в другой сесии, что документ выглядит по-прежнему:
rs0:PRIMARY> // { "_id" : ObjectId("5b450276dc396f534f119532"), "name" : "4" }

rs0:PRIMARY> // Сохраняем изменения
rs0:PRIMARY> session.commitTransaction()

rs0:PRIMARY> // Проверяем результат
rs0:PRIMARY> database.col.find({})
{ "_id" : ObjectId("5b45026edc396f534f11952f"), "name" : "1" }
{ "_id" : ObjectId("5b450272dc396f534f119530"), "name" : "2" }
{ "_id" : ObjectId("5b450274dc396f534f119531"), "name" : "3" }
{ "_id" : ObjectId("5b450276dc396f534f119532"), "name" : "44" }

rs0:PRIMARY> // Попробуем изменить несколько документов
rs0:PRIMARY> session.startTransaction()
rs0:PRIMARY> database.col.update({name: "44"}, {name: "42"})
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
rs0:PRIMARY> database.col.find({})
{ "_id" : ObjectId("5b45026edc396f534f11952f"), "name" : "1" }
{ "_id" : ObjectId("5b450272dc396f534f119530"), "name" : "2" }
{ "_id" : ObjectId("5b450274dc396f534f119531"), "name" : "3" }
{ "_id" : ObjectId("5b450276dc396f534f119532"), "name" : "42" }
rs0:PRIMARY> database.col.update({name: "1"}, {name: "21"})
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
rs0:PRIMARY> database.col.find({})
{ "_id" : ObjectId("5b45026edc396f534f11952f"), "name" : "21" }
{ "_id" : ObjectId("5b450272dc396f534f119530"), "name" : "2" }
{ "_id" : ObjectId("5b450274dc396f534f119531"), "name" : "3" }
{ "_id" : ObjectId("5b450276dc396f534f119532"), "name" : "42" }
rs0:PRIMARY> session.commitTransaction()

rs0:PRIMARY> // Проверяем результат
rs0:PRIMARY> database.col.find({})
{ "_id" : ObjectId("5b45026edc396f534f11952f"), "name" : "21" }
{ "_id" : ObjectId("5b450272dc396f534f119530"), "name" : "2" }
{ "_id" : ObjectId("5b450274dc396f534f119531"), "name" : "3" }
{ "_id" : ObjectId("5b450276dc396f534f119532"), "name" : "42" }

rs0:PRIMARY> // А теперь убедимся, что работает отмена изменений
rs0:PRIMARY> session.startTransaction()
rs0:PRIMARY> database.col.update({name: "21"}, {name: "1"})
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
rs0:PRIMARY> database.col.find({})
{ "_id" : ObjectId("5b45026edc396f534f11952f"), "name" : "1" }
{ "_id" : ObjectId("5b450272dc396f534f119530"), "name" : "2" }
{ "_id" : ObjectId("5b450274dc396f534f119531"), "name" : "3" }
{ "_id" : ObjectId("5b450276dc396f534f119532"), "name" : "42" }

rs0:PRIMARY> // Отменим изменения
rs0:PRIMARY> session.abortTransaction()
rs0:PRIMARY> database.col.find({})
{ "_id" : ObjectId("5b45026edc396f534f11952f"), "name" : "21" }
{ "_id" : ObjectId("5b450272dc396f534f119530"), "name" : "2" }
{ "_id" : ObjectId("5b450274dc396f534f119531"), "name" : "3" }
{ "_id" : ObjectId("5b450276dc396f534f119532"), "name" : "42" }

rs0:PRIMARY> // Отлично! Данные вернулись в прежнее состояние!
rs0:PRIMARY>
```

Таким образом, совершенно не напрягаясь, можно попробовать монговские транзакции уже сейчас без запуска многосерверного кластера. Советую заглянуть в [документацию](https://docs.mongodb.com/master/core/transactions/) и прочитать об ограничениях транзакций. Например о том, что транзакции &laquo;живут&raquo; не более 1 минуты, если не успеть сохранить изменения, они будут отменены.

Успехов!
