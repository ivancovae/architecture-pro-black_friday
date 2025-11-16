# Задание 8. Выявление и устранение «горячих» шардов MongoDB

- Разработайте набор метрик, чтобы отслеживать состояние шардов.
- Предложите механизмы автоматического перераспределения данных.

## Задачи

- Определять перегруженные шарды и чанки  
- Обеспечивать автоматическое перераспределение данных  
- Внедрить систему мониторинга и оповещений  
- Предусмотреть возможность изменения шард-ключа  

## 1. Метрики мониторинга

### По узлам
- `opcounters.query|insert|update|delete`  
- `opLatencies.reads|writes` p50/p95/p99  
- CPU, RAM, IOPS, сеть  
- `wiredTiger.cache.used %`, evictions/s  
- `locks.*.acquireWaitCount/Time`  
- `connections.current`, `queuedReaders/Writers`

### По коллекциям
- нагрузка по namespace (`mongotop`, `serverStatus`)  
- scanned vs returned  
- `system.profile` по `category:"Электроника"`

### Шардирование
- число чанков на шарде  
- jumbo‑чанки  
- `moveChunk` success/fail  
- очереди миграций  

### Алерты
- QPS на шарде >50% от медианы ≥10 мин  
- p95 read >500 мс ≥10 мин  
- CPU >90% ≥10 мин  
- WiredTiger cache >95%  
- >50% чанков «Электроника» на одном шарде  

---

## 2. Выявление «горячих» шардов

```javascript
sh.status()
sh.balancerStatus()

use config
db.chunks.aggregate([
  { $match: { ns: "shop.products" } },
  { $group: { _id: "$shard", cnt: { $sum: 1 } } }
])

db.chunks.find({ ns:"shop.products", jumbo:true })
```

```javascript
use shop
db.setProfilingLevel(1, { slowms: 100 })
db.system.profile.find({ "command.filter.category": "Электроника" })
```

---

## 3. Оперативные меры

```javascript
sh.setBalancerState(true)
sh.setCollectionBalancing("shop.products", true)
sh.setBalancerWindow("01:00","06:00")

sh.splitFind("shop.products", { category:"Электроника", productId:"X123" })
sh.moveChunk("shop.products", { category:"Электроника", productId:MinKey }, "shard02")
sh.mergeChunks("shop.products",
  { category:"Электроника", productId:MinKey },
  { category:"Электроника", productId:MaxKey })
```

---

## 4. Zone sharding

```javascript
sh.addShardToZone("shard01","elecA")
sh.addShardToZone("shard02","elecB")

sh.updateZoneKeyRange(
 "shop.products",
 { category:"Электроника", productId:NumberLong("-9e18") },
 { category:"Электроника", productId:NumberLong("0") },
 "elecA"
)
sh.updateZoneKeyRange(
 "shop.products",
 { category:"Электроника", productId:NumberLong("0") },
 { category:"Электроника", productId:NumberLong("9e18") },
 "elecB"
)
```

---
