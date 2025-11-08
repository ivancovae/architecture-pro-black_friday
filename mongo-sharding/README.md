# Задание 2
Для запуска проекта выполняем следующие команды:

1) запуск контейнеров:

```shell
docker-compose -f compose.yaml up -d
```

2) инициализация сервера конфигурации:

```shell
docker exec -it config-srv mongosh --port 27017 
```

далее выполняется команда:

```shell
rs.initiate(
  {
    _id : "config-server",
       configsvr: true,
    members: [
      { _id : 0, host : "config-srv:27017" }
    ]
  }
);
```
```shell
exit();
```

3) инициализация шардов:

шард №1

```shell
docker exec -it shard-1 mongosh --port 27018
```
```shell
rs.initiate(
    {
      _id : "shard-1",
      members: [
        { _id : 0, host : "shard-1:27018" },
      ]
    }
);
```
```shell
exit();
```

шард №2

```shell
docker exec -it shard-2 mongosh --port 27019
```
```shell
rs.initiate(
    {
      _id : "shard-2",
      members: [
        { _id : 1, host : "shard-2:27019" }
      ]
    }
);
```
```shell
exit();
```

4) Инициализация роутера:

```shell
docker exec -it mongo-sharding mongosh --port 27020
```
```shell
sh.addShard( "shard-1/shard-1:27018");
sh.addShard( "shard-2/shard-2:27019");
```

5) создание БД и документа:

```shell
sh.enableSharding("somedb");
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } );
```

6) заполнение тестовыми данными:

```shell
use somedb;
for(var i = 0; i < 1000; i++) db.helloDoc.insertOne({age:i, name:"ly"+i})
```
```shell
exit();
```

ТЕСТИРОВАНИЕ РАБОТОСПОСОБНОСТИ

1) Проверка работы сервиса осуществляется через swagger:

![screenshot_1.png](screenshots%2Fscreenshot_1.png)

screenshot_1.png - swagger-страница приложения pymongo_api

При отправке GET запроса на http://localhost:8086/ в postman получается ответ

```json
{
    "mongo_topology_type": "Sharded",
    "mongo_replicaset_name": null,
    "mongo_db": "somedb",
    "read_preference": "Primary()",
    "mongo_nodes": [
        [
            "mongo-sharding",
            27020
        ]
    ],
    "mongo_primary_host": null,
    "mongo_secondary_hosts": [],
    "mongo_is_primary": true,
    "mongo_is_mongos": true,
    "collections": {
        "helloDoc": {
            "documents_count": 1000
        }
    },
    "shards": {
        "shard-1": "shard-1/shard-1:27018",
        "shard-2": "shard-2/shard-2:27019"
    },
    "cache_enabled": false,
    "status": "OK"
}
```

![screenshot_2.png](screenshots%2Fscreenshot_2.png)

screenshot_2.png - результат выполнения запроса


Видно, что имеются два шарда (**shard-1**, **shard-2**), а общее количество записей **1000**.

При отправке запросов post

![screenshot_3.png](screenshots%2Fscreenshot_3.png)

общее количество увеличивается

2) Проверка успешности шардирования:

Проверка общего количества записей через команды:

```shell
docker exec -it mongo-sharding mongosh --port 27020
```
```shell
use somedb;
db.helloDoc.countDocuments();
```
Общее количество - **1002**

![screenshot_4.png](screenshots%2Fscreenshot_4.png)

screenshot_4.png - общее количество записей в БД

Проверка количества записей в шарде **shard-1**

```shell
docker exec -it shard-1 mongosh --port 27018
```
```shell
use somedb;
db.helloDoc.countDocuments();
```

Количество **492**

![screenshot_5.png](screenshots%2Fscreenshot_5.png)

screenshot_5.png - количество записей в shard-1


Проверка количества записей в шарде **shard-2**

```shell
docker exec -it shard-2 mongosh --port 27019
```
```shell
use somedb;
db.helloDoc.countDocuments();
```

Количество **510**

![screenshot_6.png](screenshots%2Fscreenshot_6.png)

screenshot_6.png - количество записей в shard-2

Шардирование работает штатно
