# Задание 10. Cassandra

## 10.1. Данные
- **Вне Cassandra (ACID):** платежи, финансы, каталог товаров.  
- **В Cassandra:** история заказов, позиции заказов (проекции), корзины (TTL), сессии (TTL), инвентарные события (журнал), витрины «заказы по дням».  

---

## 10.2. Модель

| Таблица                              | Partition key              | Clustering                | Назначение          |
|--------------------------------------|----------------------------|---------------------------|---------------------|
| `orders_by_user`                     | `user_id`                  | `order_ts DESC, order_id` | заказы пользователя |
| `order_items_by_order`               | `order_id`                 | `line_no`                 | позиции заказа      |
| `order_events_by_order`              | `order_id`                 | `event_ts DESC`           | история событий     |
| `cart_by_user`                       | `user_id`                  | `sku`                     | корзина с TTL       |
| `sessions_by_user`/`session_by_id`   | `user_id`/`session_id`     | `session_id`              | сессии с TTL        |
| `inventory_events_by_sku_day_bucket` | `(sku, day_bucket, shard)` | `event_ts DESC`           | инвентарные события |
| `orders_by_day_bucket`               | `(day_bucket, shard)`      | `order_ts DESC`           | заказы за день      |

```sql
CREATE TABLE commerce.orders_by_user (
  user_id uuid,
  order_ts timestamp,
  order_id uuid,
  status text,
  total_amount decimal,
  PRIMARY KEY ((user_id), order_ts, order_id)
) WITH CLUSTERING ORDER BY (order_ts DESC);
```

---

## 10.3. Целостность

- **Hinted Handoff:** везде.  
- **Read Repair:** включён частично (заказы, события); отключён/минимален (корзины, сессии, инвентарь).  
- **Anti-Entropy Repair:** планово (ежедневно для инвентаря/активных диапазонов, реже для TTL-данных).  

**Consistency level:**
- Заказы/события → `LOCAL_QUORUM`.  
- Корзина/сессии → `LOCAL_ONE`.  
- Инвентарь → `LOCAL_QUORUM` или `LOCAL_ONE` (при дубле в шине).  
