# DOMjudge Docker Setup (cgroup v2 compatible)

## Table of Contents
- Requirements
- Docker Installation
- Network Setup
- MariaDB Setup
- DOMserver Setup
- Admin Login
- Judgehost Setup
- Scaling Judgehosts
- Debug Guide
- Running Section (Post-Installation Testing)

---

## 0. System Requirements

Check system:

uname -r
stat -fc %T /sys/fs/cgroup

Expected:
- Kernel 5.x+
- cgroup2fs

Check Docker cgroup version:

docker info | grep -i cgroup

Expected:
- Cgroup Version: 2

---

## 1. Install Docker

sudo apt update
sudo apt install docker.io docker-compose -y
sudo systemctl enable docker
sudo systemctl start docker

Add user:

sudo usermod -aG docker $USER
newgrp docker

Test:

docker run hello-world

---

## 2. Create Docker Network

docker network create domjudge

Verify:

docker network ls

---

## 3. Start MariaDB

docker run -d   --name mariadb   --network domjudge   -e MYSQL_ROOT_PASSWORD=rootpass   -e MYSQL_DATABASE=domjudge   -e MYSQL_USER=domjudge   -e MYSQL_PASSWORD=domjudgepass   --restart unless-stopped   mariadb:10.5

Check:

docker ps
docker logs mariadb

---

## 4. Start DOMserver

docker run -d   --name domserver   --network domjudge   -p 12345:80   -e MYSQL_HOST=mariadb   -e MYSQL_ROOT_PASSWORD=rootpass   -e MYSQL_DATABASE=domjudge   -e MYSQL_USER=domjudge   -e MYSQL_PASSWORD=domjudgepass   --restart unless-stopped   domjudge/domserver:latest

Access:
http://localhost:12345

Logs:
docker logs domserver

---

## 5. Admin Password

docker exec domserver cat /opt/domjudge/domserver/etc/initial_admin_password.secret

---

## 6. Judgehost Password

docker exec domserver cat /opt/domjudge/domserver/etc/restapi.secret

---

## 7. Start Judgehost

docker run -d   --name judgehost-0   --network domjudge   --privileged   --cgroupns=host   -v /sys/fs/cgroup:/sys/fs/cgroup   -e DAEMON_ID=0   -e DOMSERVER_BASEURL=http://domserver/   -e JUDGEDAEMON_PASSWORD=<REST_API_SECRET>   --restart unless-stopped   domjudge/judgehost:latest

Logs:
docker logs -f judgehost-0

---

## 8. Scaling Judgehosts

docker run -d   --name judgehost-1   --network domjudge   --privileged   --cgroupns=host   -v /sys/fs/cgroup:/sys/fs/cgroup   -e DAEMON_ID=1   -e DOMSERVER_BASEURL=http://domserver/   -e JUDGEDAEMON_PASSWORD=<SAME_SECRET>   --restart unless-stopped   domjudge/judgehost:latest

---

## 9. Debug Guide

docker ps
docker network ls
docker logs domserver
docker logs mariadb
docker logs judgehost-0

---

## Running Section (Post-Installation Testing)

Check containers:

docker ps

Open UI:
http://localhost:12345

Login:
admin + initial_admin_password.secret

Create contest:
Admin -> Contests -> Create

Add problem and team

Test submission:

#include <stdio.h>
int main() {
    int a,b;
    scanf("%d %d",&a,&b);
    printf("%d\n",a+b);
}

Check:
Submissions, Scoreboard, Judgehosts

---

If issues occur:
Check logs of all containers
