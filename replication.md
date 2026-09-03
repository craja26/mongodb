I am planning to set up 3 node replication. 
Oplog MongoDB replica set lo options ni record chestundhi. Existing standalone MongoDB ni **single-node replica** ga conver cheyyaali.

**Current configuration backup
Take config backup before make changes.
```bash
sudo cp /etc/mongod.conf /etc/mongod.conf.before-replset

ls -l /etc/mongod.conf /etc/mongod.conf.before-replset
```
**Check the config structure**
```bash
sudo grep -nE '^(storage:|systemLog:|net:|security:|replication:|  [a-zA-Z])' /etc/mongod.conf
```
`/etc/mongod.conf` file lo `security` section lo `keyFile` and `replication:` section lo, `replSetName:` value add cheyyali.

```YAML
security:
  authorization: enabled
  keyFile: /etc/mongodb/mongodb-keyfile

replication:
  replSetName: rs0
```

**Config validation**
Validate the config before restarting MongoDB
```bash
sudo mongod --config /etc/mongod.conf --configExpand none --help >/dev/null
echo $?
```
Expected: `0`

**Restart MongoDB**
```bash
sudo systemctl restart mongod

# immediately status check:
sudo systemctl status mongod --no-pager
```
**`rs0` initialize**
Once service restart ayyaka, mongo server ki login ayyi below command run cheyyaali.
```bash
mongosh --authenticationDatabase admin -u admin -p
```
```JavaScript
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "127.0.0.1:27017" }
  ]
})
```
Expected response roughly: `{ ok: 1 }`.
Initialization successful ayyaka prompt usually: `rs0 [direct: primary] test>`
Okkosari `secondary` ani chupiste, wait few seconds and run below command
```JavaScript
rs.status().members.map(m => ({
  name: m.name,
  stateStr: m.stateStr,
  health: m.health
}))
```
Expected ga eventually:
```
[
  {
    name: "127.0.0.1:27017",
    stateStr: "PRIMARY",
    health: 1
  }
]
```
`PRIMARY` ani vachaaka prompt kuda: `rs0 [direct: primary] test>`

**Check the `rs.status()` command**
Single-node replica set successfully initialized and elected PRIMARY.
Key lines:
```
set: 'rs0'
myState: 1
votingMembersCount: 1
writableVotingMembersCount: 1
```
And most important:
```
stateStr: 'PRIMARY'
health: 1
self: true
```
**Oplog review**
`mongosh` lo  `local` database ki switch
```Javascript
use local
```
Then:
```JavaScript
db.oplog.rs.find().sort({$natural:-1}).limit(5).pretty()
```
`local` database lo `oplog.rs` collection vuntaadhi. Andhulo replica set options anni record avuthaayi.

**Check actual database changes in oplog**
Insert a document in `ecommerce` database
```JavaScript
use ecommerce

db.customers.insertOne({
  customerId: "CUST004",
  name: "Suresh Kumar",
  email: "suresh@example.com",
  city: "Berlin",
  status: "active"
})
```
Expected output: 
```JavaScript
{
  acknowledged: true,
  insertedId: ObjectId('6a98a49e3cb05e5d7e3e4a98')
}
```
Identify the INSERT operation in Oplog
Run below script in `local` database
```JavaScript
db.oplog.rs.find({
  "op": "i",
  "o._id": ObjectId("6a98a49e3cb05e5d7e3e4a98")
}).pretty()
```
Roughly ilanti output vastaadhi:
```
{
  op: "i",
  ns: "ecommerce.customers",
  ui: ...,
  o: {
    _id: ObjectId("6a98a49e3cb05e5d7e3e4a98"),
    customerId: "CUST004",
    ...
  },
  ts: Timestamp(...),
  wall: ISODate(...)
}
```
Important point:

- `op: 'i'` → Insert operation
- `ns: 'ecommerce.customers'` → ecommerce database lo customers collection
- `o` → actual document that was inserted
- `ts` → oplog timestamp
- `wall` → operation jarigina actual time

**`ts` VS `wall`**
Our output:
```
ts:   Timestamp({ t: 1788388510, i: 1 })
wall: ISODate('2026-09-02T22:35:10.626Z')
```
`wall` human-readable time.
`ts` oplog's internal timestamp, which MongoDB uses for ordering/recovery purposes.

`i: 1` ante aa timestamp value lo increment component.

**Next: UPDATE operation**
Ippudu same `CUST004` ni update cheddam. Appudu oplog lo `op: "u"` ela untundo chuddam.

ecommerce database ki switch ayi:
```Javascript
use ecommerce

db.customers.updateOne(
  { customerId: "CUST004" },
  { $set: { status: "inactive", city: "Hamburg" } }
)
```
**Find UPDATE entry in Oplog**
Switch to `local` database, run:
```javascript
db.oplog.rs.find({
  op: "u",
  ns: "ecommerce.customers",
  "o2._id": ObjectId("6a98a49e3cb05e5d7e3e4a98")
}).sort({$natural: -1}).limit(1).pretty()
```
Important points:
`op: 'u'` → Update operation.
`o2._id` → Exactly ye document update ayyindo identify chestundi.
`o.diff.u` → Em fields update ayyayo chupistundi.
```
city   → Hamburg
status → inactive
```
So original document:
```
city: Berlin
status: active
```

** DELETE entry details in Oplog**
Switch to `ecommerce` database
```javascript
use ecommerce

db.customers.deleteOne({
  customerId: "CUST004"
})
```
Expected: 
```
acknowledged: true
deletedCount: 1
```
**Verifying DELETE oplog entry**
```javascript
use local

db.oplog.rs.find({
  op: "d",
  ns: "ecommerce.customers",
  "o._id": ObjectId("6a98a49e3cb05e5d7e3e4a98")
}).sort({$natural: -1}).limit(1).pretty()
```
**Sample DELETE oplog entry**
```javascript
[
  {
    lsid: {
      id: UUID('45cd79a4-1493-41f2-b289-0b4c86ca941a'),
      uid: Binary.createFromBase64('O0CMtIVItQN4IsEOsJdrPL8s7jv5xwh5a/A5Qfvs2A8=', 0)
    },
    txnNumber: Long('2'),
    op: 'd',
    ns: 'ecommerce.customers',
    ui: UUID('17c0f898-2e98-4725-a368-cbf576d33b76'),
    o: { _id: ObjectId('6a98a49e3cb05e5d7e3e4a98') },
    stmtId: 0,
    ts: Timestamp({ t: 1788390144, i: 1 }),
    t: Long('1'),
    v: Long('2'),
    wall: ISODate('2026-09-02T23:02:24.380Z'),
    prevOpTime: { ts: Timestamp({ t: 0, i: 0 }), t: Long('-1') }
  }
]
```
Meaning:
- `op: 'd'` → Delete operation
- `ns` → `ecommerce.customers` collection
- `o._id` → delete chesina document identifier
- `ts` → oplog ordering timestamp
- `wall` → actual wall-clock time: 2026-09-02T23:02:24.380Z

Important PITR point
Actual production-grade Point-in-Time Recovery generally:
```
BASE BACKUP
    ↓
Oplog changes continuously capture
    ↓
Accidental DELETE
    ↓
Restore base backup
    ↓
Replay oplog until target timestamp
    ↓
STOP BEFORE bad DELETE
```
Mana existing database ni mess cheyyakunda, separate database create chesi PITR cheddaam.
Create a fresh database and run below commands.
```javascript
use pitr_lab

db.customers.insertMany([
  {
    customerId: "PITR001",
    name: "Ravi Kumar",
    email: "pitr.ravi@example.com",
    balance: 1000,
    status: "active"
  },
  {
    customerId: "PITR002",
    name: "Anita Sharma",
    email: "pitr.anita@example.com",
    balance: 2000,
    status: "active"
  },
  {
    customerId: "PITR003",
    name: "Vikram Reddy",
    email: "pitr.vikram@example.com",
    balance: 3000,
    status: "active"
  }
])

exit
```
**Take Base backup**
Create backup directories
```bash
sudo mkdir -p /backup/mongodb/pitr
sudo chown -R mongod:mongod /backup/mongodb/pitr
```
Run backup command
```bash
sudo -u mongod mongodump \
  --authenticationDatabase admin \
  -u admin \
  --db pitr_lab \
  --out /backup/mongodb/pitr/base_$(date +%Y%m%d_%H%M%S)

# check the Backup directory
sudo ls -ltrh /backup/mongodb/pitr/
```
Capture current timestamp in local database
```javascript
use local

rs0 [direct: primary] local> db.oplog.rs.find().sort({$natural:-1}).limit(1).pretty()
[
  {
    op: 'n',
    ns: '',
    o: { msg: 'periodic noop' },
    ts: Timestamp({ t: 1788445982, i: 1 }),
    t: Long('2'),
    v: Long('2'),
    wall: ISODate('2026-09-03T14:33:02.085Z')
  }
```
**Create a change after backup**
```javascript
use pitr_lab

db.customers.updateOne(
  { customerId: "PITR001" },
  { $set: { balance: 1500 } }
)
```
**Update oplog entry**
`mongosh` lo `local` database ki switch ayyi:
```javascript
use local

db.oplog.rs.find({
  op: "u",
  ns: "pitr_lab.customers",
  "o2.customerId": "PITR001"
}).sort({$natural: -1}).limit(1).pretty()
```
Verify oplog entry
```javascript
use local

rs0 [direct: primary] local> db.oplog.rs.find({
|   op: "u",
|   ns: "pitr_lab.customers",
|   "o2._id": ObjectId("6a9983c5c75ca614cc84ff56")
| }).sort({$natural: -1}).limit(1).pretty()
[
  {
    lsid: {
      id: UUID('049bf937-fa61-4ed1-8c97-15479727751b'),
      uid: Binary.createFromBase64('O0CMtIVItQN4IsEOsJdrPL8s7jv5xwh5a/A5Qfvs2A8=', 0)
    },
    txnNumber: Long('1'),
    op: 'u',
    ns: 'pitr_lab.customers',
    ui: UUID('aca25b47-189a-4ea8-a899-a173ed78bf69'),
    o: { '$v': 2, diff: { u: { balance: 1500 } } },
    o2: { _id: ObjectId('6a9983c5c75ca614cc84ff56') },
    stmtId: 0,
    ts: Timestamp({ t: 1788446161, i: 1 }),
    t: Long('2'),
    v: Long('2'),
    wall: ISODate('2026-09-03T14:36:01.240Z'),
    prevOpTime: { ts: Timestamp({ t: 0, i: 0 }), t: Long('-1') }
  }
]
```
**Apply another change**
```javascript
use pitr_lab

db.customers.updateOne(
  { customerId: "PITR002" },
  { $set: { balance: 2500 } }
)
```
Checking oplog entry
```javascript
use local

db.oplog.rs.find({
  op: "u",
  ns: "pitr_lab.customers",
  "o2._id": ObjectId("6a9983c5c75ca614cc84ff57")
}).sort({$natural: -1}).limit(1).pretty()
[
  {
    lsid: {
      id: UUID('782f7f8a-c525-4da0-b021-af26bbe6af27'),
      uid: Binary.createFromBase64('O0CMtIVItQN4IsEOsJdrPL8s7jv5xwh5a/A5Qfvs2A8=', 0)
    },
    txnNumber: Long('1'),
    op: 'u',
    ns: 'pitr_lab.customers',
    ui: UUID('aca25b47-189a-4ea8-a899-a173ed78bf69'),
    o: { '$v': 2, diff: { u: { balance: 2500 } } },
    o2: { _id: ObjectId('6a9983c5c75ca614cc84ff57') },
    stmtId: 0,
    ts: Timestamp({ t: 1788446769, i: 1 }),
    t: Long('2'),
    v: Long('2'),
    wall: ISODate('2026-09-03T14:46:09.968Z'),
    prevOpTime: { ts: Timestamp({ t: 0, i: 0 }), t: Long('-1') }
  }
]
```
Insert another document:
```javascript
use pitr_lab

db.customers.insertOne({
  customerId: "PITR004",
  name: "Suresh Kumar",
  email: "pitr.suresh@example.com",
  balance: 4000,
  status: "active"
})
```
Checking oplog entry
```javascript
use local

db.oplog.rs.find({
  op: "i",
  ns: "pitr_lab.customers",
  "o._id": ObjectId("6a99894c565170a9b7c93e48")
}).sort({$natural: -1}).limit(1).pretty()
[
  {
    lsid: {
      id: UUID('049bf937-fa61-4ed1-8c97-15479727751b'),
      uid: Binary.createFromBase64('O0CMtIVItQN4IsEOsJdrPL8s7jv5xwh5a/A5Qfvs2A8=', 0)
    },
    txnNumber: Long('2'),
    op: 'i',
    ns: 'pitr_lab.customers',
    ui: UUID('aca25b47-189a-4ea8-a899-a173ed78bf69'),
    o: {
      _id: ObjectId('6a99894c565170a9b7c93e48'),
      customerId: 'PITR004',
      name: 'Suresh Kumar',
      email: 'pitr.suresh@example.com',
      balance: 4000,
      status: 'active'
    },
    o2: { _id: ObjectId('6a99894c565170a9b7c93e48') },
    stmtId: 0,
    ts: Timestamp({ t: 1788447052, i: 2 }),
    t: Long('2'),
    v: Long('2'),
    wall: ISODate('2026-09-03T14:50:52.888Z'),
    prevOpTime: { ts: Timestamp({ t: 0, i: 0 }), t: Long('-1') }
  }
]
```
**Accidental DELETE**
```javascript
use pitr_lab

db.customers.deleteOne({
  customerId: "PITR004"
})
```
Find oplog DELETE command entry
```javascript
rs0 [direct: primary] pitr_lab> use local
switched to db local
rs0 [direct: primary] local> db.oplog.rs.find({
|   op: "d",
|   ns: "pitr_lab.customers",
|   "o._id": ObjectId("6a99894c565170a9b7c93e48")
| }).sort({$natural: -1}).limit(1).pretty()
[
  {
    lsid: {
      id: UUID('049bf937-fa61-4ed1-8c97-15479727751b'),
      uid: Binary.createFromBase64('O0CMtIVItQN4IsEOsJdrPL8s7jv5xwh5a/A5Qfvs2A8=', 0)
    },
    txnNumber: Long('3'),
    op: 'd',
    ns: 'pitr_lab.customers',
    ui: UUID('aca25b47-189a-4ea8-a899-a173ed78bf69'),
    o: { _id: ObjectId('6a99894c565170a9b7c93e48') },
    stmtId: 0,
    ts: Timestamp({ t: 1788447333, i: 1 }),
    t: Long('2'),
    v: Long('2'),
    wall: ISODate('2026-09-03T14:55:33.059Z'),
    prevOpTime: { ts: Timestamp({ t: 0, i: 0 }), t: Long('-1') }
  }
]
```
**Recovery Target**
Suppose production incident ila jarigindhi:
| **“14:55:33.059 UTC ki accidental DELETE jarigindi. DELETE ki just mundu unna database state ni recover cheyyali.”**
Appudu target point roughly:
```
14:55:33.059 UTC
       ↑
       │
STOP BEFORE DELETE
```
So recovery result lo:

- PITR001 → balance 1500 ✅
- PITR002 → balance 2500 ✅
- PITR003 → balance 3000 ✅
- PITR004 → present ✅
- DELETE operation → replay cheyyakudadhu ❌









































