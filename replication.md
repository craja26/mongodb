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



























