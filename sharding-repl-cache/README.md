# Задание 4

Обновленный файл контейнеров: [compose.yaml](compose.yaml) 
выполнить команды:

```shell
docker-compose up redis -d 
```
```shell
docker-compose up -d
```

Для проверки скорости запроса и ответа использовал программу **postman**. Выполнил запрос *GET http://localhost:8086/helloDoc/users*
Результат представлен на фиг. 9

![screeenshot_4_1.png](img%2Fscreeenshot_4_1.png)

screeenshot_4_1.png - результат первого запроса

Время запроса/ответа составило **1,05 с**

Повторный запрос:

![screeenshot_4_2.png](img%2Fscreeenshot_4_2.png)

screeenshot_4_2.png - результат второго запроса

Время запроса/ответа составило **15 мс**