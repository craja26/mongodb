# MongoDB 8.0 - Three-Node replica set

## 1. Objective

Build a 3-node MongoDB Replica Set on Rocky Linux 8 VMs and practice:

- MongoDB installation
- Dedicated database disk
- Linux/OS configuration
- SELinux configuration
- Keyfile authentication
- Replica Set initialization
- PRIMARY / SECONDARY roles
- Replication
- Secondary reads and Read Preference
- PRIMARY failure and automatic election
- Recovery and rejoining
- Replica Set member priority
- `rs.stepDown()`
- Priority takeover

---

# 2. Architecture

Replica set name:

```text
rsLab
```
Servers:

| Hostname | IP | MongoDB | Initial Role | Priority |
|---|---|---|---|---:|
| mongo02 | 192.168.2.132 | 8.0.29 | PRIMARY | 2 |
| mongo03 | 192.168.2.133 | 8.0.29 | SECONDARY | 1 |
| mongo04 | 192.168.2.134 | 8.0.29 | SECONDARY | 1 |

Later, for the priority experiment:

```text
mongo02 → priority 2
mongo03 → priority 3
mongo04 → priority 1
```

---

# 3. VM Preparation

Created three Rocky Linux 8 VMs:

```text
mongo02
mongo03
mongo04
```

Configured bridged networking so that all VMs could communicate with each other.

Verified hostnames:

```bash
hostname
```

Verified IP addresses:

```bash
ip addr
```

Configured timezone:

```bash
sudo timedatectl set-timezone Europe/Berlin
```

Verified:

```bash
timedatectl
```

---

# 4. Dedicated MongoDB Data Disk

Added a second 30 GB disk to each VM.

Verified disks:

```bash
lsblk
```

Created an XFS filesystem on the second disk:

```bash
sudo mkfs.xfs /dev/sdb1
```

Created MongoDB data directory:

```bash
sudo mkdir -p /data/db
```

Mounted the disk:

```bash
sudo mount /dev/sdb1 /data/db
```

Verified:

```bash
df -h /data/db
```

Added the filesystem to `/etc/fstab` using the UUID.

Verified the mount:

```bash
mount | grep /data/db
```

Changed ownership:

```bash
sudo chown mongod:mongod /data/db
```

---

# 5. OS Tuning

Created:

```text
/etc/sysctl.d/99-mongodb.conf
```

Configuration:

```text
vm.swappiness = 1
vm.overcommit_memory = 1
vm.max_map_count = 131060
```

Applied:

```bash
sudo sysctl --system
```

Verified:

```bash
sysctl vm.swappiness
sysctl vm.overcommit_memory
sysctl vm.max_map_count
```

Note:

Transparent Huge Pages (THP) were not manually disabled for this MongoDB 8.0 lab.

---

# 6. MongoDB Repository

Configured the official MongoDB 8.0 repository.

Example repository:

```ini
[mongodb-org-8.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/8/mongodb-org/8.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://pgp.mongodb.com/server-8.0.asc
```

Refresh repository metadata:

```bash
sudo dnf clean all
sudo dnf repolist
```

---

# 7. MongoDB Installation

Installed MongoDB:

```bash
sudo dnf install -y mongodb-org
```

Verified version:

```bash
mongod --version
```

All three nodes were running:

```text
MongoDB 8.0.29
```

---

# 8. SELinux Configuration

Installed SELinux management utilities:

```bash
sudo dnf install -y policycoreutils-python-utils
```

Checked the existing MongoDB SELinux context:

```bash
ls -Zd /var/lib/mongo
```

The custom `/data/db` directory initially did not have the correct MongoDB SELinux context.

Added a persistent SELinux rule:

```bash
sudo semanage fcontext -a -t mongod_var_lib_t "/data/db(/.*)?"
```

Applied the context:

```bash
sudo restorecon -Rv /data/db
```

Verified:

```bash
ls -Zd /data/db
```

Also verified the persistent rule:

```bash
sudo semanage fcontext -l | grep '/data/db'
```

---

# 9. Hostname Resolution

Added all Replica Set members to `/etc/hosts`:

```text
192.168.2.132   mongo02.lab.local mongo02
192.168.2.133   mongo03.lab.local mongo03
192.168.2.134   mongo04.lab.local mongo04
```

Verified hostname resolution:

```bash
getent hosts mongo02
getent hosts mongo03
getent hosts mongo04
```

Verified SSH connectivity:

```bash
ssh rockylinux@mongo04 hostname
```

---

# 10. Firewall

Opened MongoDB port:

```bash
sudo firewall-cmd --permanent --add-port=27017/tcp
sudo firewall-cmd --reload
```

Verified:

```bash
sudo firewall-cmd --list-ports
```

---

# 11. MongoDB Keyfile Authentication

Created a fresh keyfile for this Replica Set on `mongo02`.

```bash
sudo mkdir -p /etc/mongodb

sudo openssl rand -base64 756 | \
sudo tee /etc/mongodb/mongodb-keyfile > /dev/null
```

Changed ownership:

```bash
sudo chown mongod:mongod /etc/mongodb/mongodb-keyfile
```

Restricted permissions:

```bash
sudo chmod 400 /etc/mongodb/mongodb-keyfile
```

The same keyfile was distributed to `mongo03` and `mongo04`.

Important:

All Replica Set members must use the **same keyfile**.

Do not commit the actual keyfile to GitHub.

---

# 12. MongoDB Configuration

Backed up the original configuration:

```bash
sudo cp /etc/mongod.conf /etc/mongod.conf.before-rsLab
```

## mongo02

```yaml
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

storage:
  dbPath: /data/db

processManagement:
  timeZoneInfo: /usr/share/zoneinfo

net:
  port: 27017
  bindIp: 127.0.0.1,mongo02

security:
  authorization: enabled
  keyFile: /etc/mongodb/mongodb-keyfile

replication:
  replSetName: rsLab
```

## mongo03

Same configuration, except:

```yaml
bindIp: 127.0.0.1,mongo03
```

## mongo04

Same configuration, except:

```yaml
bindIp: 127.0.0.1,mongo04
```

Important configuration detail:

`timeZoneInfo` belongs under:

```yaml
processManagement:
```

not under:

```yaml
storage:
```

---

# 13. Start MongoDB

Enabled MongoDB:

```bash
sudo systemctl enable mongod
```

Started MongoDB:

```bash
sudo systemctl start mongod
```

Checked status:

```bash
sudo systemctl status mongod --no-pager
```

Verified port:

```bash
sudo ss -lntp | grep 27017
```

Each node was listening on its own IP and localhost.

---

# 14. Replica Set Initialization

Initially MongoDB was running in Replica Set mode but had not yet been initialized.

`rs.status()` showed that no Replica Set configuration had been received.

Connected to `mongo02` and initiated the Replica Set:

```javascript
rs.initiate({
  _id: "rsLab",
  members: [
    { _id: 0, host: "mongo02:27017", priority: 2 },
    { _id: 1, host: "mongo03:27017", priority: 1 },
    { _id: 2, host: "mongo04:27017", priority: 1 }
  ]
})
```

---

# 15. Verify Replica Set

```javascript
rs.status()
```

Initial healthy state:

```text
mongo02 → PRIMARY
mongo03 → SECONDARY
mongo04 → SECONDARY
```

The Replica Set had:

```text
votingMembersCount: 3
majorityVoteCount: 2
```

This means a majority of two voting members is required for majority-based operations.

---

# 16. Create Administrator User

On the PRIMARY:

```javascript
use admin

db.createUser({
  user: "admin",
  pwd: passwordPrompt(),
  roles: [
    { role: "root", db: "admin" }
  ]
})
```

The user was created successfully.

Important:

The administrator user is stored in the Replica Set data and does not need to be manually created on every node.

---

# 17. Test Replication

Connected to the PRIMARY:

```bash
mongosh --host mongo02 --port 27017 -u admin -p --authenticationDatabase admin
```

Created test database and collection:

```javascript
use labdb

db.test.insertMany([
  { _id: 1, name: "Raja", role: "DBA" },
  { _id: 2, name: "MongoDB", role: "Database" },
  { _id: 3, name: "ReplicaSet", role: "DBRE" }
])
```

Verified:

```javascript
db.test.countDocuments()
```

Expected:

```text
3
```

---

# 18. Read Data from SECONDARY

Connected directly to `mongo03`:

```bash
mongosh --host mongo03 --port 27017 -u admin -p --authenticationDatabase admin
```

Switched database:

```javascript
use labdb
```

Default read preference is `primary`.

Therefore, reading directly from a SECONDARY may produce a not-primary/read-preference error.

For this lab, explicitly selected secondary reads:

```javascript
db.getMongo().setReadPref("secondary")
```

Then:

```javascript
db.test.find().sort({ _id: 1 })
```

The same data was successfully read from `mongo03`.

Repeated the test on `mongo04` and received the same data.

---

# 19. Read Preference Concept

MongoDB separates:

```text
Replica Set Role
        +
Client Read Preference
```

Typical behavior:

```text
PRIMARY + primary read preference
    → read succeeds

SECONDARY + default primary read preference
    → read is not directed to that secondary

SECONDARY + secondary read preference
    → read can be served by the secondary
```

Important:

Secondary reads can potentially return slightly stale data because replication is asynchronous.

For critical read-after-write operations, primary reads are generally preferred.

---

# 20. PRIMARY Failure Test

Current state:

```text
mongo02 → PRIMARY
mongo03 → SECONDARY
mongo04 → SECONDARY
```

Stopped MongoDB on `mongo02`:

```bash
sudo systemctl stop mongod
```

From another Replica Set member, checked:

```javascript
rs.status().members.map(m => ({
  name: m.name,
  state: m.stateStr,
  health: m.health
}))
```

Observed:

```text
mongo02 → (not reachable/healthy)
mongo03 → PRIMARY
mongo04 → SECONDARY
```

MongoDB automatically detected the PRIMARY failure and elected another eligible member.

---

# 21. Write After Failover

Connected to the new PRIMARY (`mongo03`):

```bash
mongosh --host mongo03 --port 27017 -u admin -p --authenticationDatabase admin
```

Then:

```javascript
use labdb

db.test.insertOne({
  _id: 4,
  name: "FailoverTest",
  role: "DBRE"
})
```

Verified:

```javascript
db.test.find().sort({ _id: 1 })
```

The new PRIMARY accepted writes.

Then checked `mongo04` and confirmed that `_id: 4` was replicated.

This demonstrated:

```text
PRIMARY failure
      ↓
Automatic election
      ↓
New PRIMARY
      ↓
New writes
      ↓
Replication continues
```

---

# 22. Restart Failed PRIMARY

Started `mongo02` again:

```bash
sudo systemctl start mongod
```

Verified:

```bash
sudo systemctl status mongod --no-pager
```

After rejoining the Replica Set, `mongo02` initially became a SECONDARY while another member was PRIMARY.

The node synchronized with the Replica Set.

---

# 23. Replica Set Priority Experiment

Original configuration:

```text
mongo02 → priority 2
mongo03 → priority 1
mongo04 → priority 1
```

Changed `mongo03` priority to 3.

On the PRIMARY:

```javascript
cfg = rs.conf()

cfg.members[1].priority = 3

rs.reconfig(cfg)
```

Verified:

```javascript
rs.conf().members.map(m => ({
  host: m.host,
  priority: m.priority
}))
```

Result:

```text
mongo02 → priority 2
mongo03 → priority 3
mongo04 → priority 1
```

---

# 24. Observe Priority-Based Election

After the configuration change, `mongo03` became PRIMARY:

```text
mongo02 → SECONDARY
mongo03 → PRIMARY
mongo04 → SECONDARY
```

This demonstrated that a healthy higher-priority member can be preferred for PRIMARY.

Important:

`priority` does not mean that a node is permanently PRIMARY.

It controls the member's preference/eligibility during elections and priority takeover behavior.

---

# 25. Controlled PRIMARY Step-Down

Connected to the current PRIMARY (`mongo03`):

```bash
mongosh --host mongo03 --port 27017 -u admin -p --authenticationDatabase admin
```

Confirmed that the shell was connected to PRIMARY:

```text
rsLab [direct: primary]
```

Executed:

```javascript
rs.stepDown(60)
```

This voluntarily removed the current PRIMARY from its PRIMARY role and made it temporarily ineligible for re-election.

Another eligible member was elected:

```text
mongo02 → PRIMARY
mongo03 → SECONDARY
mongo04 → SECONDARY
```

---

# 26. Priority Takeover Observation

Although `mongo03` had previously stepped down, its priority was still:

```text
mongo03 → priority 3
mongo02 → priority 2
mongo04 → priority 1
```

After some time, `mongo03` became PRIMARY again.

Observed state:

```text
mongo02 → SECONDARY
mongo03 → PRIMARY
mongo04 → SECONDARY
```

This demonstrated priority takeover behavior.

Conceptually:

```text
mongo03 PRIMARY
     |
     | rs.stepDown()
     ↓
mongo02 PRIMARY
mongo03 SECONDARY
     |
     | higher-priority member becomes eligible
     ↓
mongo03 PRIMARY
```

---

# 27. Final Concepts Learned

## Replica Set

A group of MongoDB instances that maintain copies of the same data and provide high availability.

## PRIMARY

The member that normally accepts writes.

## SECONDARY

A member that replicates data from another member and can serve reads when the client's read preference allows it.

## Replication

Changes written to the PRIMARY are replicated to SECONDARY members.

## Election

When the PRIMARY becomes unavailable, eligible members vote to elect a new PRIMARY.

## Priority

Controls a member's preference for becoming PRIMARY.

Higher priority means stronger preference, assuming the member is eligible and healthy.

## `rs.stepDown()`

Allows the current PRIMARY to voluntarily step down.

Useful for controlled maintenance and testing.

## Read Preference

Controls where MongoDB clients prefer to read from.

Common modes include:

```text
primary
primaryPreferred
secondary
secondaryPreferred
nearest
```

## Keyfile Authentication

Allows members of a self-managed Replica Set to authenticate with each other when internal authentication is enabled.

The keyfile must be protected and must not be committed to Git.

---

# 28. Useful Commands

Check Replica Set status:

```javascript
rs.status()
```

Check Replica Set configuration:

```javascript
rs.conf()
```

Check member states:

```javascript
rs.status().members.map(m => ({
  name: m.name,
  state: m.stateStr,
  health: m.health
}))
```

Check priorities:

```javascript
rs.conf().members.map(m => ({
  host: m.host,
  priority: m.priority
}))
```

Check MongoDB service:

```bash
sudo systemctl status mongod
```

Start MongoDB:

```bash
sudo systemctl start mongod
```

Stop MongoDB:

```bash
sudo systemctl stop mongod
```

Restart MongoDB:

```bash
sudo systemctl restart mongod
```

Check MongoDB port:

```bash
sudo ss -lntp | grep 27017
```

Check logs:

```bash
sudo tail -f /var/log/mongodb/mongod.log
```

---

# 29. Important Lab Safety Notes

Never commit these to GitHub:

```text
/etc/mongodb/mongodb-keyfile
MongoDB passwords
Private SSH keys
Secrets
API keys
Production connection strings
```

For GitHub documentation, use placeholders such as:

```text
<YOUR_PASSWORD>
<KEYFILE>
<PRIVATE_IP>
```

The lab configuration and commands can be documented, but secrets should remain outside the repository.

---

# 30. Final Architecture

```text
                         rsLab
                           |
                 ┌─────────┴─────────┐
                 │                   │
             mongo02             mongo03
             PRIMARY              SECONDARY
            priority 2            priority 3*
                 │                   │
                 └─────────┬─────────┘
                           │
                       mongo04
                      SECONDARY
                     priority 1

* Priority 3 was used during the priority experiment.
```

The lab demonstrated MongoDB high availability, replication, elections, failover, recovery, read preference, priority, and controlled PRIMARY step-down.
