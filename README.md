
---

# DOMjudge Docker Setup (cgroup v2 + Debug Guide)

This setup runs a full DOMjudge system using Docker:

* MariaDB (database)
* DOMserver (web + API)
* Judgehost (sandboxed execution, cgroup v2 compatible)

---

# 0. System Requirements (DO NOT SKIP)

## Check kernel + cgroups

```bash id="k1c9qz"
uname -r
stat -fc %T /sys/fs/cgroup
```

Expected:

* Kernel: Linux **5.x+**
* Output: `cgroup2fs`

---

## Check Docker cgroup mode

```bash id="m8x0ld"
docker info | grep -i cgroup
```

Expected:

```
Cgroup Version: 2
```

---

## If NOT cgroup v2

### Problem:

Judgehost will fail or loop restarting.

### Fix:

Enable cgroup v2 in GRUB:

```bash id="r9n2aa"
sudo nano /etc/default/grub
```

Add or edit:

```
GRUB_CMDLINE_LINUX="systemd.unified_cgroup_hierarchy=1"
```

Then:

```bash id="z3p1km"
sudo update-grub
sudo reboot
```

---

# 1. Install Docker

```bash id="d7q0tt"
sudo apt update
sudo apt install docker.io docker-compose -y
sudo systemctl enable docker
sudo systemctl start docker
```

Add user:

```bash id="u1b9ff"
sudo usermod -aG docker $USER
newgrp docker
```

---

## Debug check

```bash id="v4m1tt"
docker run hello-world
```

### If fails:

* Docker service not running
* Permission issue → re-login or reboot

---

# 2. Create Docker Network

```bash id="n8c2zz"
docker network create domjudge
```

---

## Debug

```bash id="q2k9lm"
docker network ls
```

### If missing:

* Command failed silently → rerun
* Docker daemon issue → restart docker

---

# 3. Start MariaDB

```bash id="b5x7aa"
docker run -d \
  --name mariadb \
  --network domjudge \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -e MYSQL_DATABASE=domjudge \
  -e MYSQL_USER=domjudge \
  -e MYSQL_PASSWORD=domjudgepass \
  --restart unless-stopped \
  mariadb:10.5
```

---

## Debug MariaDB

```bash id="p1d8yy"
docker ps
```

Expected:

```
mariadb   Up
```

### If restarting/exiting:

```bash id="x7q2jj"
docker logs mariadb
```

Common issues:

* Port conflict (rare)
* Corrupt volume (fix by removing container)

Fix:

```bash id="c9l1pp"
docker rm -f mariadb
```

---

# 4. Start DOMserver

```bash id="f8w2tt"
docker run -d \
  --name domserver \
  --network domjudge \
  -p 12345:80 \
  -e MYSQL_HOST=mariadb \
  -e MYSQL_ROOT_PASSWORD=rootpass \
  -e MYSQL_DATABASE=domjudge \
  -e MYSQL_USER=domjudge \
  -e MYSQL_PASSWORD=domjudgepass \
  --restart unless-stopped \
  domjudge/domserver:latest
```

---

## Debug DOMserver

Open:

```
http://localhost:12345
```

If not working:

```bash id="l9m1kk"
docker logs domserver
```

### Common issues:

* MariaDB not ready yet → wait 10–30 sec
* Wrong DB credentials
* Network issue → check domjudge network

---

# 5. Admin Password

```bash id="s3q9aa"
docker exec domserver cat /opt/domjudge/domserver/etc/initial_admin_password.secret
```

---

## Debug login issues

If login fails:

* wrong username (must be `admin`)
* copied password has newline → re-copy carefully

---

# 6. Judgehost Password

```bash id="t8c1pp"
docker exec domserver cat /opt/domjudge/domserver/etc/restapi.secret
```

---

## Debug

If file missing:

```bash id="u9x2kk"
docker exec domserver ls /opt/domjudge/domserver/etc/
```

If empty → DOMserver not initialized properly.

---

# 7. Start Judgehost (cgroup v2 critical)

```bash id="y1q8zz"
docker run -d \
  --name judgehost-0 \
  --network domjudge \
  --privileged \
  --cgroupns=host \
  -v /sys/fs/cgroup:/sys/fs/cgroup \
  -e DAEMON_ID=0 \
  -e DOMSERVER_BASEURL=http://domserver/ \
  -e JUDGEDAEMON_PASSWORD=<REST_API_SECRET> \
  --restart unless-stopped \
  domjudge/judgehost:latest
```

---

##  Judgehost Debug (VERY IMPORTANT)

### Check logs:

```bash id="z2k1pp"
docker logs -f judgehost-0
```

---

##  Common failures

### 1. Authentication failed

Cause:

* Wrong REST API password

Fix:

* re-copy `restapi.secret`

---

### 2. cgroup errors

Cause:

* missing `--cgroupns=host`

Fix:

* restart container with correct flags

---

### 3. Cannot reach domserver

Cause:

* wrong base URL

Fix:

```bash id="h5l9tt"
DOMSERVER_BASEURL=http://domserver/
```

NOT localhost.

---

#  8. Scaling Judgehosts

Add more workers:

```bash id="k2m9yy"
docker run -d \
  --name judgehost-1 \
  --network domjudge \
  --privileged \
  --cgroupns=host \
  -v /sys/fs/cgroup:/sys/fs/cgroup \
  -e DAEMON_ID=1 \
  -e DOMSERVER_BASEURL=http://domserver/ \
  -e JUDGEDAEMON_PASSWORD=<SAME_SECRET> \
  --restart unless-stopped \
  domjudge/judgehost:latest
```

---

##Scaling debug

If load doesn’t increase:

* check judgehost status in UI
* ensure different DAEMON_ID values
* check CPU usage (`top`, `docker stats`)

---

# 9. Full System Debug Checklist

If something breaks, run:

```bash id="n8p4zz"
docker ps
docker network ls
docker logs domserver
docker logs mariadb
docker logs judgehost-0
```

---

## Mental model

* MariaDB → data layer
* DOMserver → brain + UI
* Judgehost → execution engine (most fragile part)

---

# Done

You now have:

* working DOMjudge
* cgroup v2 compatible judgehost
* scalable architecture
* debugging toolkit (actually important part)

---

## Running Section (Post-Installation Testing)

*Check containers:

```
docker ps -a
```
this should be like:

```
┌──(white㉿white)-[~/Documents/DomJudge-on-CgroupV2]
└─$ docker ps -a
CONTAINER ID   IMAGE                       COMMAND                  CREATED          ... NAMES
9811199b4359   hello-world                 "/hello"                 2 minutes ago    ... reverent_johnson
bdd5f6598796   domjudge/judgehost:latest   "/usr/bin/dumb-init …"   30 minutes ago   ... judgehost-1
1038de916157   domjudge/judgehost:latest   "/usr/bin/dumb-init …"   33 minutes ago   ... judgehost-0
916a2810daec   domjudge/domserver:latest   "/scripts/start.sh"      43 minutes ago   ... domserver
34890b27e135   mariadb:10.5                "docker-entrypoint.s…"   45 minutes ago   ... mariadb
```
*Start MariaDB
```
docker start mariadb
```
*Start DomServer
```
docker start domserver
```

*Start all JudgeHosts
```
docker start judgehost-0
```
Repeat this for all judgehosts

*Open UI
```
http://localhost:12345
```

*Login
admin + initial_admin_password.secret

*Create contest
Admin -> Contests -> Create

*Add problem and team

*Test submission

```
#include <stdio.h>
int main() {
    int a,b;
    scanf("%d %d",&a,&b);
    printf("%d\n",a+b);
}
```

*Check
Submissions, Scoreboard, Judgehosts

---
