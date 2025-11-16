# Задание 9. Чтение с реплик и консистентность

## Таблица решений

|   Коллекция    |      Операция чтения                      | Требуемая       | Куда читать        | Настройки (пример)    | Допустимая задержка | Обоснование                                    |
|                |                                           | консистентность |                    |                       | репликации          |                                                |
|----------------|-------------------------------------------|-----------------|--------------------|-----------------------|---------------------|------------------------------------------------|
|  **products**  | Каталог/листинг                           | Eventual        | secondaryPreferred | readConcern: local,   | ≤ 60s               | Листинги терпят устаревание, высокая нагрузка. |
|                |                                           |                 |                    | maxStaleness: 60s     |                     |                                                |
|    products    | Страница товара                           | Eventual        | secondaryPreferred | readConcern: local,   | ≤ 60s               | Допустимо устаревание до полминуты.            |
|                | (без «в корзину»)                         |                 |                    | maxStaleness: 60s     |                     |                                                |
|    products    | Поиск/автодополнение                      | Eventual        | secondary          | readConcern: local,   | ≤ 60s               | Высокочастотные запросы, не критично.          |
|                |                                           |                 |                    | maxStaleness: 60s     |                     |                                                |
|    products    | Проверка цены/остатка при рендере корзины | Strong          | primary            | readConcern: majority | 0s                  | Риск продать по старой цене или без остатка.   |
|                |                                           |                 |                    |                       |                     |                                                |
|    products    | Витрина «новинки/хиты»                    | Eventual        | secondaryPreferred | readConcern: local,   | ≤ 60s               | Допустимо устаревание.                         |
|                |                                           |                 |                    |  maxStaleness: 60s    |                     |                                                |
|  **carts**     | Просмотр корзины                          | Causal          | primaryPreferred   | causal sessions       | ≤ 5s                | Нужен read‑your‑own‑writes.                    |
|                |                                           |                 |                    |                       |                     |                                                |
|    carts       | Проверка перед оплатой                    | Strong          | primary            | readConcern: majority | 0s                  | Должны быть точные цены и остатки.             |
|                |                                           |                 |                    |                       |                     |                                                |
|    carts       | Аналитика (заброшенные корзины)           | Eventual        | secondary          | readConcern: local,   | ≤ 120s              | BI‑нагрузка, не влияет на UX.                  |
|                |                                           |                 |                    | maxStaleness: 120s    |                     |                                                |
|  **orders**    | История заказов (список)                  | Strong enough   | primaryPreferred   | readConcern: majority | ≤ 5s                | Пользователь ждёт актуальные статусы.          |
|                |                                           |                 |                    |                       |                     |                                                |
|    orders      | Детали заказа (состав)                    | Eventual        | secondaryPreferred | readConcern: local    | ≤ 60s               | Почти не меняется после создания.              |
|                |                                           |                 |                    |                       |                     |                                                |
|    orders      | Текущий статус заказа                     | Strong          | primary            | readConcern: majority | 0s                  | Нельзя показывать устаревший статус.           |
|                |                                           |                 |                    |                       |                     |                                                |
|    orders      | Админ‑панель                              | Strong          | primary            | readConcern:          | 0s                  | Операционные решения требуют свежести.         |
|                |                                           |                 |                    | majority/linearizable |                     |                                                |
|    orders      | BI/отчёты                                 | Eventual        | secondary          | readConcern: local,   | ≤ 300s              | Отчётность, допустим лаг.                      |
|                |                                           |                 |                    | maxStaleness: 300s    |                     |                                                |

---
## Принципы

1. Критичные для денег операции → **primary**, `readConcern: majority` или `linearizable`.  
2. Листинги/поиск/BI → **secondary**, ограничение `maxStalenessSeconds`.  
3. Часто меняющиеся сущности (carts, orders.status) → primary.  
4. «Read your own writes» → causal consistency или `primaryPreferred`.  
5. Лаги:  
   - Критика: 0s-0.01s.  
   - Пользовательские списки: ≤30–60s.  
   - Аналитика: ≤300s.  
---

## Примеры

```js
// Каталог
db.products.find({}, { readPreference: "secondaryPreferred", maxStalenessSeconds: 60 });

// Статус заказа
db.orders.find({ userId, orderId }, { readPreference: "primary", readConcern: { level: "majority" } });

// Корзина с read-your-own-writes
const session = client.startSession({ causalConsistency: true });
session.withTransaction(async () => {
  await db.carts.updateOne(...);
  await db.carts.find({ userId }, { readPreference: "primaryPreferred" });
});
```