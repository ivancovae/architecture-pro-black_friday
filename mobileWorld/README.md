# MongoDB шардирование и операции

```javascript
sh.enableSharding("shop")

use shop

db.products.createIndex({ category: 1, _id: "hashed" })
sh.shardCollection("shop.products", { category: 1, _id: "hashed" })
db.products.createIndex({ category: 1, price: 1 })
db.products.createIndex({ sku: 1 }, { unique: true })
db.products.createIndex({ name: "text" })
db.products.insertOne({
  sku: "5606794",
  name: "6.67 Смартфон Xiaomi Redmi Note 14 256 ГБ",
  category: "Смартфоны",
  price: NumberDecimal("16199.00"),
  attributes: { color: "black", size: "256GB" },
  stock: { "RU-MOW": 120, "RU-EKB": 50, "RU-KGD": 30 },
  updated_at: new Date()
})
db.products.find({ category: "Смартфоны", price: { $gte: 10000, $lte: 60000 } })

db.orders.createIndex({ user_id: 1, created_at: 1 })
sh.shardCollection("shop.orders", { user_id: 1, created_at: 1 })
db.orders.createIndex({ user_id: 1, created_at: -1 })
db.orders.createIndex({ status: 1, created_at: -1 })
db.orders.insertOne({
  _id: "u:64f9c2f1aabbccddeeff0011:" + ObjectId(),
  user_id: ObjectId("64f9c2f1aabbccddeeff0011"),
  created_at: new Date(),
  status: "placed",
  total: NumberDecimal("89990.00"),
  geozone: "RU-MOW",
  items: [
    { product_id: ObjectId(), qty: 1, price: NumberDecimal("149990.00") },
    { product_id: ObjectId(), qty: 2, price: NumberDecimal("120000.00") }
  ]
})
db.orders.find({ user_id: ObjectId("64f9c2f1aabbccddeeff0011") })
         .sort({ created_at: -1 })
         .limit(10)

db.carts.createIndex({ owner_key: "hashed" })
sh.shardCollection("shop.carts", { owner_key: "hashed" })
db.carts.createIndex({ expires_at: 1 }, { expireAfterSeconds: 0 })
db.carts.createIndex(
  { owner_key: 1, status: 1 },
  { unique: true, partialFilterExpression: { status: "active" } }
)
db.carts.insertOne({
  owner_key: "u:64f9c2f1aabbccddeeff0011",
  user_id: ObjectId("64f9c2f1aabbccddeeff0011"),
  items: [ { product_id: ObjectId(), quantity: 2 } ],
  status: "active",
  created_at: new Date(),
  updated_at: new Date(),
  expires_at: new Date(Date.now() + 7*24*3600*1000)
})
db.carts.findOne({ owner_key: "u:64f9c2f1aabbccddeeff0011", status: "active" })
db.carts.updateOne(
  { owner_key: "u:64f9c2f1aabbccddeeff0011", status: "active" },
  { $push: { items: { product_id: ObjectId(), quantity: 1 } },
    $currentDate: { updated_at: true } }
)
db.carts.updateOne(
  { owner_key: "u:64f9c2f1aabbccddeeff0011", status: "active" },
  { $pull: { items: { product_id: ObjectId("64f9c2f1aabbccddeeffabcd") } },
    $currentDate: { updated_at: true } }
)

db.inventory.createIndex({ geozone: 1, product_id: 1 })
sh.shardCollection("shop.inventory", { geozone: 1, product_id: 1 })
db.inventory.insertOne({
  geozone: "RU-MOW",
  product_id: ObjectId(),
  qty: 100,
  updated_at: new Date()
})
db.inventory.updateOne(
  { geozone: "RU-MOW", product_id: ObjectId("64f9c2f1aabbccddeeffabcd"), qty: { $gte: 5 } },
  { $inc: { qty: -5 }, $set: { updated_at: new Date() } }
)
```

#Коллекции и ключи шардирования

Catalog
  *Коллекция: products
  *Shard Key: { category: 1, _id: "hashed" }
  *Индексы:
    - { category: 1, price: 1 }
    - { sku: 1 } unique
    - { name: "text" }

Order
  *Коллекция: orders
  *Shard Key: { user_id: 1, created_at: 1 }
  *Формат _id: "u:<user_id>:<ObjectId>"
  *Индексы:
    - { user_id: 1, created_at: -1 }
    - { status: 1, created_at: -1 }

Cart
  *Коллекция: carts
  *Shard Key: { owner_key: "hashed" }
  *Индексы:
    - TTL: { expires_at: 1 }
    - Partial unique: { owner_key: 1, status: 1 }
      partial: { status: "active" }
