## Sharding in MongoDB using docker

This repo is customized of repo `mongodb-sharding` of `yasasdy`:
```url
https://github.com/yasasdy/mongodb-sharding.git
```

**This repo is used to test sharding demo**

#### **Clone repo**
Clone my custom repo:
```bash
git clone https://github.com/mthuwhihand/mongodb-sharding-with-docker.git
```

Go to folder
```bash
cd mongodb-sharding-with-docker
code .
```

#### **Config Servers:**
Create network
```bash
docker network create mongo-shard-net
```

To start the config servers, execute the following Docker command:

```bash
docker-compose -f config_server/docker-compose.yaml up -d
```

Once the instances are up, connect to the container using the following command:

```bash
mongosh mongodb://localhost:10001
```

Inside the container, pass the instances as members to form a replica set:
```bash
rs.initiate({
  _id: "config_rs",
  configsvr: true,
  members: [
    { _id: 0, host: "configsvr1:27017" },
    { _id: 1, host: "configsvr2:27017" },
    { _id: 2, host: "configsvr3:27017" }
  ]
});
```

Check the replica set status using the following command:

```bash
rs.status()
```

##### **Shard Servers**

Repeat a similar process for creating shard-1 and shard-2 Docker containers:

```bash
docker-compose -f shard_server1/docker-compose.yaml up -d 
``` 

```bash
docker-compose -f shard_server2/docker-compose.yaml up -d
```

Initiate the replica sets for shard-1 and shard-2:

**In shard-1:**

```bash
mongosh mongodb://localhost:20001  
```

```bash
rs.initiate({
  _id: "shard1_rs",
  members: [
    { _id: 0, host: "shardsvr1_1:27017" },
    { _id: 1, host: "shardsvr1_2:27017" },
    { _id: 2, host: "shardsvr1_3:27017" }
  ]
});
```

Check the replica set status using the following command:
```bash
rs.status()
```

Exit command
```bash
exit
```

**In shard-2:**

```bash
mongosh mongodb://localhost:20004
```

```bash
rs.initiate({
  _id: "shard2_rs",
  members: [
    { _id: 0, host: "shardsvr2_1:27017" },
    { _id: 1, host: "shardsvr2_2:27017" },
    { _id: 2, host: "shardsvr2_3:27017" }
  ]
});
```

Check the replica set status using the following command:
```bash
rs.status()
```

Exit command
```bash
exit
```

# 3. Mongo Routers

Finally, start the Mongo routers:

```bash
docker-compose -f mongo_router/docker-compose.yaml up -d
```

```bash
mongosh mongodb://localhost:30000
```

Inside the container, add both shards to the cluster:

```bash
sh.addShard("shard1_rs/shardsvr1_1:27017,shardsvr1_2:27017,shardsvr1_3:27017");
```

```bash
sh.addShard("shard2_rs/shardsvr2_1:27017,shardsvr2_2:27017,shardsvr2_3:27017");
```

# Sharding the Collection

Ensure your application is always connected to Mongo routers; connecting directly to shard replica sets is not recommended.

Connect to the Mongo router:
```bash
mongosh mongodb://localhost:30000
```
Run this command to create a database and turn on sharding for db and collection:
```bash
use ordersdb;
sh.enableSharding("ordersdb");

db.orders.createIndex({ customer_id: "hashed" })
sh.shardCollection("ordersdb.orders", { customer_id: "hashed" })
```

Check shards:
```bash
sh.status()
```

Check documents of each shard
```bash
db.orders.aggregate([
  { $collStats: { storageStats: {} } },
  { $project: { shard: "$shard", count: "$storageStats.count" } }
])
```
Check shard data:
```bash
db.orders.aggregate([
  { $collStats: { storageStats: {} } },
  { $project: { shard: 1, size: "$storageStats.size", count: "$storageStats.count" } }
])
```
Check chunks:
```bash
use config
```

```bash
db.chunks.aggregate([
  { $match: { ns: "ordersdb.orders" } },
  { $group: { _id: "$shard", chunks: { $sum: 1 } } }
])
```

Add random 1 milion orders:
```bash
function getRandomProduct() {
  const products = ["Laptop", "Phone", "Tablet", "Monitor", "Keyboard", "Mouse"];
  return products[Math.floor(Math.random() * products.length)];
}

function getRandomQuantity() {
  return Math.floor(Math.random() * 10) + 1;
}

function getRandomDate() {
  const start = new Date(2022, 0, 1);
  const end = new Date();
  return new Date(start.getTime() + Math.random() * (end.getTime() - start.getTime()));
}

let bulk = db.orders.initializeUnorderedBulkOp();
const batchSize = 10000;

for (let i = 1; i <= 1000000; i++) {
  bulk.insert({
    _id: ObjectId(),
    customer_id: ObjectId(),
    product: getRandomProduct(),
    quantity: getRandomQuantity(),
    order_date: getRandomDate()
  });

  if (i % batchSize === 0) {
    bulk.execute();
    print(`Inserted ${i} documents`);
    bulk = db.orders.initializeUnorderedBulkOp();
  }
}

// Insert
if (bulk.length > 0) {
  bulk.execute();
  print("Inserted remaining documents");
}

```

```bash
db.orders.countDocuments()
```
Down containers
```bash
docker-compose -f config_server/docker-compose.yaml down -v
```

```bash
docker-compose -f shard_server1/docker-compose.yaml down -v
``` 

```bash
docker-compose -f shard_server2/docker-compose.yaml down -v
```

```bash
docker-compose -f mongo_router/docker-compose.yaml down -v
```

### References:
```url
https://github.com/yasasdy/mongodb-sharding.git
```

```url
https://medium.com/@yasasvi/mongodb-sharding-with-docker-c8b18bee32eb
```
