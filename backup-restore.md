### Backup
**Create directory and grant access to mongod user**
```bash
sudo chown -R mongod:mongod /backup/mongodb
ls -ld /backup/mongodb
```
Run `mongodump` as a `mongod` user
```bash
sudo -u mongod mongodump \
  --authenticationDatabase admin \
  -u admin \
  --db ecommerce \
  --out /backup/mongodb/ecommerce_$(date +%Y%m%d_%H%M%S)
```
Need to enter admin user password.

#### Restore
Restore database with different name
```bash
sudo -u mongod mongorestore \
  --authenticationDatabase admin \
  -u admin \
  --nsFrom='ecommerce.*' \
  --nsTo='ecommerce_restore_test.*' \
  /backup/mongodb/ecommerce_20260902_213801
```
Ikkada `--nsFrom` and `--nsTo` use chestunna. Ante backup database `ecommerce` and restore database `ecommerce_restore_test`.


### Restore single collection
Accidental ga single-collection drop ayite, below command run chesi restore cheyyochu.
```bash
sudo -u mongod mongorestore \
  --authenticationDatabase admin \
  -u admin \
  --db ecommerce \
  --collection customers \
  /backup/mongodb/ecommerce_20260902_213801/ecommerce/customers.bson
```

### Compress Backup

```bash
sudo mkdir -p /backup/mongodb/gzip
sudo chown mongod:mongod /backup/mongodb/gzip

sudo -u mongod mongodump \
  --authenticationDatabase admin \
  -u admin \
  --db ecommerce \
  --gzip \
  --out /backup/mongodb/gzip/ecommerce_$(date +%Y%m%d_%H%M%S)
```

### Restore compressed backup
```bash
sudo -u mongod mongorestore \
  --authenticationDatabase admin \
  -u admin \
  --gzip \
  --nsFrom='ecommerce.*' \
  --nsTo='ecommerce_gzip_restore.*' \
  /backup/mongodb/gzip/ecommerce_20260902_234015
```


