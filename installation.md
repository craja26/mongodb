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
### First MongoDB startup
Before starting, one last quick check:
```bash
sudo ls -ldZ /data/db
```
We expect:
```text
mongod mongod ... mongod_var_lib_t ... /data/db
```
If that looks correct, start MongoDB:
```bash
sudo systemctl start mongod
```
Then immediately:
```bash
sudo systemctl status mongod --no-pager
```
**Expected**

Something like:
```text
Active: active (running)
```
If it says failed, don't try random fixes.
Immediately run:
```bash
sudo journalctl -u mongod -n 50 --no-pager
```
and:
```bash
sudo tail -50 /var/log/mongodb/mongod.log
```
### Verify MongoDB health
First run:
```bash
mongosh --quiet --eval 'db.adminCommand({ping:1})'
```
Expected:
```text
{ ok: 1 }
```
Then:
```bash
mongosh --quiet --eval 'db.serverStatus().version'
```
Expected:
```text
8.0.31
```
Now verify the actual database path:
```bash
mongosh --quiet --eval 'db.serverCmdLineOpts().parsed.storage.dbPath'
```
Expected:
```text
/data/db
```
**Then check listening port**
```bash
sudo ss -lntp | grep 27017
```
Because currently our config still has:
```YAML
bindIp: 127.0.0.1
port: 27017
```
So we expect MongoDB to listen only on localhost.
That's actually good for now because we haven't configured authentication/network security yet.
**Finally check the MongoDB log for startup errors**
```bash
sudo tail -30 /var/log/mongodb/mongod.log
```
**Lets check MongoDB process limits.** Earlier we saw the systemd unit explicitly has:
```text
LimitNOFILE=64000
LimitNPROC=64000
LimitMEMLOCK=infinity
TasksMax=infinity
```
Now let's verify what the running PID 3143 actually received.
Run:
```bash
sudo cat /proc/3143/limits | egrep 'open files|max user processes|max locked memory'
```
And:
```bash
sudo systemctl is-enabled mongod
```
Expected:
```text
enabled
```
This is useful because we're not just saying "systemd config says 64000" — we're checking the actual running process.

## MongoDB Security
Current state:
```text
MongoDB
  ├── Running
  ├── localhost only
  └── Authentication disabled
```
Authentication disabled unna situation lo admin user create cheyyadam first.
**⚠️ Important order**
```text
1. Create admin user
        ↓
2. Verify admin login
        ↓
3. Enable authorization
        ↓
4. Restart MongoDB
        ↓
5. Verify authenticated access
```
### Create admin user
Run:
```bash
mongosh
```
You should enter the MongoDB shell:
```text
test>
```
Now execute:
```JavaScript
use admin
```
Then:
```JavaScript
db.createUser({
  user: "admin",
  pwd: passwordPrompt(),
  roles: [
    { role: "root", db: "admin" }
  ]
})
```
MongoDB password prompt adugutundi. Password type chestunnappudu screen lo characters kanipinchavu — that's normal.
If successful, MongoDB something like:
```text
{ ok: 1 }
```
Then exit:
```JavaScript
exit
```
### Test admin authentication BEFORE enabling auth
This step very important.
Run:
```bash
mongosh --authenticationDatabase admin -u admin -p
```
It will ask password. Enter the password you just created. If successful:
```text
test>
```
Then run:
```JavaScript
db.runCommand({ connectionStatus: 1 })
```
### Enable MongoDB authorization
First, current config backup already undi, but auth enable cheyyadaniki mundu fresh backup teesukundam:
```bash
sudo cp -p /etc/mongod.conf /etc/mongod.conf.pre-auth
```
Now edit:
```bash
sudo vi /etc/mongod.conf
```
Current section:
```YAML
#security:
```
ni:
```YAML
security:
  authorization: enabled
```
ga change chesi, save cheyyaali.

**Verify config**
Run:
```bash
grep -A2 '^security:' /etc/mongod.conf
```
Expected:
```text
security:
  authorization: enabled
```
Then:
```bash
sudo systemctl restart mongod
```
Immediately check:
```bash
sudo systemctl status mongod --no-pager
```
Expected:
```text
Active: active (running)
```
### Test that authentication is actually enforced
First without credentials:
```bash
# mongosh --quiet --eval 'db.adminCommand({ping:1})' # ping lo authentication check cheyyadu.
# database information access lo credentials adugutundhi. Lets try below command.
# mongosh --quiet --eval 'db.adminCommand({connectionStatus:1})'
mongosh --quiet --eval 'db.adminCommand({listDatabases:1})'
mongosh --quiet --eval 'db.getSiblingDB("admin").getUsers()'

```
This time we expect an authentication error, something like:
```text
MongoServerError: Command ping requires authentication
```
That's good. 😎
Then authenticate:
```bash
mongosh --authenticationDatabase admin -u admin -p
```
Enter your password.
Inside:
```JavaScript
# db.adminCommand({ping:1})
db.getSiblingDB("admin").getUsers()
```
Expected:
```text
{
  users: [
    {
      _id: 'admin.admin',
      userId: UUID('4fda91ef-7342-48ca-8326-48d27dd5cb70'),
      user: 'admin',
      db: 'admin',
      roles: [ { role: 'root', db: 'admin' } ],
      mechanisms: [ 'SCRAM-SHA-1', 'SCRAM-SHA-256' ]
    }
  ],
  ok: 1
}
```
Then exit.

















































