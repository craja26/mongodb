###MongoDB PITR — Complete Hands-on Lab Notes

####1. PITR ante enti?

**PITR = Point-In-Time Recovery**
Database ni complete backup teesina time ki matrame restore cheyyadam kaakunda, backup tarvata jarigina changes ni oplog nunchi replay chesi, specific time/state varaku recover cheyyadam.
Example:
```
21:51  → Baseline backup
21:52  → BASE001 = 1500
21:54  → BASE002 = 2500
21:55  → BASE004 inserted
21:56  → BASE004 deleted

             ↓

Target recovery point = just BEFORE 21:56 delete

             ↓

Recovered:
BASE001 = 1500
BASE002 = 2500
BASE003 = 3000
BASE004 = 4000
```
---

####2. Lab database create cheyyadam
Primary MongoDB instance lo:
```bash
mongosh --authenticationDatabase admin -u admin -p
```
Database select:
```javascript
use pitr_lab2
```
Collection create + baseline data:
```javascript
db.customers.insertMany([
  {
    customerId: "BASE001",
    name: "Ravi Kumar",
    balance: 1000,
    status: "active"
  },
  {
    customerId: "BASE002",
    name: "Anita Sharma",
    balance: 2000,
    status: "active"
  },
  {
    customerId: "BASE003",
    name: "Vikram Reddy",
    balance: 3000,
    status: "active"
  }
])
```
Verify:
```javascript
db.customers.find().sort({customerId:1})
```
Expected:
```
BASE001 → 1000
BASE002 → 2000
BASE003 → 3000
```
---
####3. Baseline backup create cheyyadam
Important: PITR lab kosam full instance dump with `--oplog` use chesam.
```bash
sudo mkdir -p /backup/mongodb/pitr_lab2_final

sudo chown -R mongod:mongod \
  /backup/mongodb/pitr_lab2_final
```
Backup:
```bash
sudo -u mongod mongodump \
  --authenticationDatabase admin \
  -u admin \
  --oplog \
  --out /backup/mongodb/pitr_lab2_final/full_$(date +%Y%m%d_%H%M%S)
```
Example: `/backup/mongodb/pitr_lab2_final/full_20260903_215115`
Important concept

`--oplog` entire historical oplog ni backup cheyyadu.

It captures the oplog entries generated during the dump, mainly to make that backup internally consistent.

For true arbitrary PITR, we need continuous oplog capture/retention.

---
####4. Controlled database changes create cheyyadam
Baseline backup ayyaka changes create cheddam.
**Change 1 — BASE001 update**
```javascript
use pitr_lab2

db.customers.updateOne(
  {customerId:"BASE001"},
  {$set:{balance:1500}}
)
```
Oplog timestamp record chesam: `Timestamp({t:1788465178,i:1})`

---
**Change 2 — BASE002 update**
```javascript
db.customers.updateOne(
  {customerId:"BASE002"},
  {$set:{balance:2500}}
)
```
Oplog: `Timestamp({t:1788465257,i:1})`

---
**Change 3 — BASE004 insert**
```javascript
db.customers.insertOne({
  customerId:"BASE004",
  name:"Suresh Rao",
  balance:4000,
  status:"active"
})
```
Oplog: `Timestamp({t:1788465316,i:1})`

---
**Change 4 — BASE004 delete**
```javascript
db.customers.deleteOne({
  customerId:"BASE004"
})
```
Oplog: `Timestamp({t:1788465383,i:1})`

---
####5. Oplog entries identify cheyyadam
Primary MongoDB lo:
```javascript
use local

db.oplog.rs.find({
  ns: "pitr_lab2.customers",
  ts: {
    $gte: Timestamp({t:1788465178,i:1}),
    $lte: Timestamp({t:1788465383,i:1})
  }
}).sort({ts:1}).pretty()
```
Manaki 4 entries vachayi:
```
1. UPDATE BASE001
2. UPDATE BASE002
3. INSERT BASE004
4. DELETE BASE004
```
---
####6. Recovery target decide cheyyadam
Manaki target:
| BASE004 delete cheyyadaniki just BEFORE unna database state kavali.
Therefore delete oplog timestamp: `1788465383:1`
Target: `before 1788465383:1`
So replay cheyyalsinavi:
```
BASE001 update  ✅
BASE002 update  ✅
BASE004 insert  ✅
BASE004 delete  ❌
```
####7. Required oplog entries export cheyyadam
Recovery oplog directory:
```bash
sudo mkdir -p /backup/mongodb/pitr_lab2_final/oplog

sudo chown -R mongod:mongod \
  /backup/mongodb/pitr_lab2_final/oplog
```
Export:
```bash
sudo -u mongod mongodump \
  --authenticationDatabase admin \
  -u admin \
  --db local \
  --collection oplog.rs \
  --query '{"ns":"pitr_lab2.customers","ts":{"$gte":{"$timestamp":{"t":1788465178,"i":1}},"$lte":{"$timestamp":{"t":1788465383,"i":1}}}}' \
  --out /backup/mongodb/pitr_lab2_final/oplog
```
Expected: `done dumping local.oplog.rs (4 documents)`
Copy it as the recovery oplog:
```bash
sudo cp \
  /backup/mongodb/pitr_lab2_final/oplog/local/oplog.rs.bson \
  /backup/mongodb/pitr_lab2_final/recovery_oplog.bson
```
Mana actual lab lo manam idi: `/backup/mongodb/pitr_lab2_final/recovery/oplog.bson` ga maintain chesam.

---
####8. Separate recovery MongoDB instance create cheyyadam
Original MongoDB instance ni disturb cheyyakunda port 27018 lo separate recovery instance use chesam.
Directory:
```bash
sudo mkdir -p /data/pitr_recovery
sudo chown mongod:mongod /data/pitr_recovery
sudo chmod 700 /data/pitr_recovery
```
Log directory:
```bash
sudo mkdir -p /var/log/mongodb/pitr_recovery

sudo chown mongod:mongod \
  /var/log/mongodb/pitr_recovery
```

---
####9. Recovery mongod start cheyyadam
```bash
sudo -u mongod mongod \
  --dbpath /data/pitr_recovery \
  --port 27018 \
  --bind_ip 127.0.0.1 \
  --logpath /var/log/mongodb/pitr_recovery/mongod.log \
  --fork
```
Check:
```bash
mongosh --port 27018 --quiet \
  --eval 'db.adminCommand({ping:1})'
```
Expected: `{ ok: 1 }`

---
####10. Recovery database clean ga undho check cheyyadam
```bash
mongosh --port 27018 --quiet --eval '
db = db.getSiblingDB("pitr_lab2");
print("documents =", db.customers.countDocuments())
'
```
Expected: `documents = 0`

---
####11. Baseline backup restore + PITR oplog replay
**Important point**
Baseline dump directory lo original `oplog.bson` unte, and separate `--oplogFile` provide chesthe conflict vastundi.
So original baseline `oplog.bson` ni temporarily rename chesam:
```bash
sudo mv \
  /backup/mongodb/pitr_lab2_final/full_20260903_215115/oplog.bson \
  /backup/mongodb/pitr_lab2_final/full_20260903_215115/oplog.bson.original
```
Then actual PITR command:
```bash
mongorestore \
  --port 27018 \
  --oplogReplay \
  --oplogLimit 1788465383:1 \
  --oplogFile /backup/mongodb/pitr_lab2_final/recovery/oplog.bson \
  /backup/mongodb/pitr_lab2_final/full_20260903_215115
```
What happens?
First: `Baseline restore`
So: 
```
BASE001 = 1000
BASE002 = 2000
BASE003 = 3000
```
Then:
```
replaying oplog
applied 3 oplog entries
```
Why 3?
```
BASE001 update   → replayed ✅
BASE002 update   → replayed ✅
BASE004 insert   → replayed ✅
BASE004 delete   → skipped ❌
```
Because: `--oplogLimit 1788465383:1`
delete timestamp ki mundu unna entries replay ayyayi; delete itself replay kaaledu.

---
####12. Final PITR verification
```bash
mongosh --port 27018 --quiet --eval '
db = db.getSiblingDB("pitr_lab2");
db.customers.find(
  {},
  {_id:0,customerId:1,name:1,balance:1,status:1}
).sort({customerId:1}).forEach(printjson);
print("count =", db.customers.countDocuments());
'
```
Final result:
```
BASE001 → 1500
BASE002 → 2500
BASE003 → 3000
BASE004 → 4000

count = 4
```
🎯 PITR SUCCESS

---

####13. Entire flow
Mama, idi important. Short ga ila cheppachu:
```
1. Take a consistent baseline backup.
2. Retain/capture oplog continuously.
3. Identify the required recovery timestamp.
4. Restore the baseline backup to a separate recovery instance.
5. Apply oplog entries sequentially up to the target timestamp.
6. Stop before the unwanted transaction.
7. Validate the recovered data.
8. Redirect application traffic after validation.
```
Simple architecture
```
                    PRODUCTION
                        │
                        │
              ┌─────────┴─────────┐
              │                   │
         Full Backup            Oplog
              │              Continuous capture
              │                   │
              └─────────┬─────────┘
                        │
                        ▼
                Recovery Server
                  Port 27018
                        │
             Restore baseline
                        │
                        ▼
                 Replay oplog
                        │
                        ▼
              Target Point in Time
                        │
                        ▼
                 Validate data
```
One very important real-world point

Mana lab lo manual oplog extraction use chesi PITR concept demonstrate chesam. Production environment lo continuous backup/oplog management ni properly automate cheyyali;
MongoDB's managed/backup tooling is designed for this use case.

Notes lo main takeaway:
|PITR = Baseline backup + continuous oplog + target timestamp + sequential oplog replay.
