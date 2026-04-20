
---

# DOMjudge Docker Setup (cgroup v2 compatible)

This guide sets up a full DOMjudge environment using Docker:

* MariaDB
* DOMserver
* Judgehost (cgroup v2 compatible)

---

# 0. Requirements

Check system compatibility:

```bash
uname -r
stat -fc %T /sys/fs/cgroup
```

Expected:

* Kernel: Linux 5.x+
* cgroup: `cgroup2fs`

Check Docker cgroup version:

```bash
docker info | grep -i cgroup
```

Expected:

```
Cgroup Version: 2
```

---

# 1. Install Docker

```bash
sudo apt update
sudo apt install docker.io docker-compose -y
sudo systemctl enable docker
sudo systemctl start docker
```

Add user to docker group:

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Test:

```bash
docker run hello-world
```

---

# 2. Create Docker Network

```bash
docker network create domjudge
```

Verify:

```bash
docker network ls
```

---

# 3. Start MariaDB

```bash
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

Check:

```bash
docker ps
```

---

# 4. Start DOMserver

```bash
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

Access:

```
http://localhost:12345
```

---

# 5. Get Admin Password

```bash
docker exec domserver cat /opt/domjudge/domserver/etc/initial_admin_password.secret
```

Login:

* user: `admin`
* password: output of above command

---

# 6. Get Judgehost Password

```bash
docker exec domserver cat /opt/domjudge/domserver/etc/restapi.secret
```

Save this — required for judgehost authentication.

---

# 7. Start Judgehost (cgroup v2 required)

```bash
docker run -d \
  --name judgehost-0 \
  --network domjudge \
  --privileged \
  --cgroupns=host \
  -v /sys/fs/cgroup:/sys/fs/cgroup \
  -e DAEMON_ID=0 \
  -e DOMSERVER_BASEURL=http://domserver/ \
  -e JUDGEDAEMON_PASSWORD=<PASTE_RESTAPI_SECRET> \
  --restart unless-stopped \
  domjudge/judgehost:latest
```

---

# 8. Scaling Judgehosts

To improve performance, add more judgehosts:

```bash
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

Change only:

* `--name judgehost-X`
* `DAEMON_ID=X`

---

# 9. Verify System

Check containers:

```bash
docker ps
```

Check judgehost logs:

```bash
docker logs -f judgehost-0
```

Expected:

```
Judge started
```

---

# Common Issues

### Judgehost not connecting

* Wrong `JUDGEDAEMON_PASSWORD`
* DOMserver not reachable
* Missing `--cgroupns=host`

### Database errors

* MariaDB not running
* Wrong credentials

---

# Done

You now have:

* DOMjudge server
* Database backend
* Working judgehost
* Scalable judging system

---

