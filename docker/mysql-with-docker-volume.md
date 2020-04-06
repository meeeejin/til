# How to install MySQL and use a Docker volume on Docker

## Reference

- [Get Docker Engine - Community for Ubuntu](https://docs.docker.com/install/linux/docker-ce/ubuntu/)
- [DockerHub - MySQL](https://hub.docker.com/_/mysql)
- [Docker Compose](https://github.com/docker/compose)

## Install Docker Engine

1. Update the `apt` package index:

```bash
$ sudo apt-get update
```

2. Install the necessary packages to allow `apt` to use a repository over HTTPS:

```bash
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```

3. Add Docker’s official GPG key:

```bash
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

Verify that you now have the key with the fingerprint:

```bash
$ sudo apt-key fingerprint 0EBFCD88
    
pub   rsa4096 2017-02-22 [SCEA]
      9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
uid           [ unknown] Docker Release (CE deb) <docker@docker.com>
sub   rsa4096 2017-02-22 [S]
```

4. Set up the stable repository:

```bash
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

5. Install the latest version of Docker Engine:

```bash
$ sudo apt-get update
$ sudo apt-get install docker-ce docker-ce-cli containerd.io
```

### Test Docker Installation

1. Verify that Docker Engine - Community is installed correctly by running the hello-world image:

```bash
$ sudo docker run hello-world

Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
ca4f61b1923c: Pull complete
Digest: sha256:ca0eeb6fb05351dfc8759c20733c91def84cb8007aa89a5bf606bc8b315b9fc7
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
...
```

2. Run `docker image ls` to list the hello-world image that you downloaded to your machine:

```bash
$ sudo docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
hello-world         latest              fce289e99eb9        15 months ago       1.84kB
```

3. You can check the status of all the containers using the below command:

```bash
$ sudo docker ps -a
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                     PORTS                               NAMES
2eeed464cd9f        hello-world         "/hello"                 2 minutes ago       Exited (0) 2 minutes ago                                       objective_perlman
89bb80313ac4        mysql:5.7           "docker-entrypoint.s…"   15 minutes ago      Up 15 minutes              0.0.0.0:3306->3306/tcp, 33060/tcp   vldb-mysql
```

4. You can stop one or more running containers:

```bash
$ sudo docker stop [container-name]
```

## Install MySQL

### Via a Simple Command

Start a MySQL instance is simple:

```bash
$ sudo docker run --name some-mysql -e MYSQL_ROOT_PASSWORD=pw -d mysql:5.7
```

- `some-mysql`: The name you want to assign to your container
- `pw`: The password to be set for the MySQL root user
- `mysql:5.7`: The tag specifying the MySQL version you want. See the [manual](https://hub.docker.com/_/mysql) for relevant tags.

### Via `docker-compose`

You can use `docker-compose` to run a MySQL container. Define the services that make up your app in `docker-compose.yml` so they can be run together in an isolated environment. For example, my `dockers-compose.yml` looks like this:

```bash
version: '3.1'

services:
  db:
    image: mysql:5.7
    container_name: vldb-mysql
    ports:
      - 3306:3306
    volumes:
      - /home/mijin/test_data:/var/lib/mysql
      - /home/mijin/cnf:/etc/mysql/conf.d
    environment:
      MYSQL_ROOT_PASSWORD: "pw"
```

Running `docker-compose up` makes *Compose* start and run the MySQL app:

```bash
$ sudo docker-compose up -d
Creating vldb-mysql ... done
```

### Use a Directory of Host System as a Volume

The mounted directory in the host system can be used as a data directory on MySQL in Docker. Add `volumes` in the *yaml* file (`dockers-compose.yml`):

```bash
  db:
    image: mysql:5.7
    ...
    volumes:
      - /home/mijin/test_data:/var/lib/mysql
    ...
```

- `/home/mijin/test_data`: The SSD-mounted directory in host system
- `/var/lib/mysql`: The data directory for MySQL in Docker 

### Add Custom my.cnf to MySQL Container

We can also map a customized `my.cnf` to MySQL container.

1. Create a new `my.cnf` file:

```bash
$ vim /home/mijin/mysql-conf/my.cnf
...
```

2. Modify `dockers-compose.yml` to map `mysql-conf` directory in host system into `conf.d` directory in Docker:

```bash
  db:
    image: mysql:5.7
    ...
    volumes:
      - /home/mijin/mysql-conf:/etc/mysql/conf.d
    ...
```

3. Run `docker-compose up`:

```bash
$ sudo docker-compose up -d
Creating vldb-mysql ... done
```

4. Run the below command to connect to the created container's bash:

```bash
$ sudo docker exec -it vldb-mysql bash
```

5. Check the modified server variable in MySQL:

```bash
root@89bb80313ac4:/# mysql -uroot -p -e "show variables like '%log_files%'"
+---------------------------+-------+
| Variable_name             | Value |
+---------------------------+-------+
| innodb_log_files_in_group | 3     |
+---------------------------+-------+
```

The value of `innodb_log_files_in_group` was changed from 2 (default value) to 3 (the value set in `mysql-conf/my.cnf`).