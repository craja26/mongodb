Before installing, one important thing: MongoDB 8.0 package repository officially supports RHEL 8-compatible systems, including Rocky Linux 8. We'll use the official MongoDB repository rather than random RPM downloads.
### Create MongoDB repository
First let's make sure the machine can reach MongoDB's repository:
```bash
curl -I https://repo.mongodb.org/
```
Expected something like:
```text
Expected something like:
```
### Add MongoDB 8.0 repository
Create repo file
```bash
sudo vi /etc/yum.repos.d/mongodb-org-8.0.repo
```
Paste the below text
```ini
[mongodb-org-8.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/8/mongodb-org/8.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://pgp.mongodb.com/server-8.0.asc
```
### Verify YUM sees the repository
Run:
```bash
sudo yum repolist
```
You should see something similar to:
```
mongodb-org-8.0
```
Then:
```bash
sudo yum info mongodb-org
```
This is important because installation before checking the available package/version is bad DBA practice. First verify what repository is actually offering.

### Install MongoDB
Run
```bash
sudo yum install -y mongodb-org
```
This will install the MongoDB server plus the standard tools and systemd service files.
After installation **don't start MongoDB yet.**
First verify the installed packages:
```bash
rpm -qa | grep '^mongodb-org' | sort
```
And:
```bash
mongod --version
```
### 🛑 Important — don't start mongod yet
We still need to configure:
```text
/etc/mongod.conf
       ↓
dbPath: /data/db
       ↓
logging
       ↓
bindIp
       ↓
port
       ↓
security/authentication
```
### Inspect default MongoDB configuration
Run:
```bash
sudo cat /etc/mongod.conf
```
And:
```bash
sudo systemctl status mongod --no-pager
```
And verify MongoDB user:
```bash
id mongod
```
Finally data directory:
```bash
ls -ld /data/db
```
### First check SELinux context
First compare:
```bash
ls -Zd /var/lib/mongo
```
And:
```bash
ls -Zd /data/db
```
Also:
```bash
sudo semanage fcontext -l | grep -E 'mongod|mongo' | head -20
```
**Backup configuration**
This is safe production habit. We can always roll back.
```bash
sudo cp -p /etc/mongod.conf /etc/mongod.conf.bak
```
Then verify:
```bash
ls -l /etc/mongod.conf*
```
**Create persistent SELinux mapping**
`/var/lib/mongo ` ki vunna exact MongoDB type: `mongod_var_lib_t` ni `/data/db` ki assign cheddam
Run:
```bash
sudo semanage fcontext -a -t mongod_var_lib_t "/data/db(/.*)?"
```
**What does this do?**
Idi actual files ni immediate ga relabel cheyyadu.

It creates a persistent rule:
```text
/data/db/*
      ↓
mongod_var_lib_t
```
`restorecon ` / full SELinux relabel ayyaka correct contect maintain avuthundhi/
**Apply the SELinux context**
Now:
```bash
sudo restorecon -Rv /data/db
```
**Verify persistent SELinux rule**
```bash
sudo semanage fcontext -l | grep '/data/db'
```
**Then ownership change**
MongoDB process `mongod` user tho run avtundi. So:
```bash
sudo chown -R mongod:mongod /data/db
```
Then verify:
```bash
ls -ld /data/db
```
And SELinux context should remain:
```bash
ls -Zd /data/db
```
### Configure `mongod.conf`
Ippudu MongoDB default:
```YAML
storage:
  dbPath: /var/lib/mongo
```















































