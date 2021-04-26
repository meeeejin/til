# Installing Percona Monitoring and Management (PMM) on Ubuntu

Reference
- [Get Docker CE for Ubuntu](https://docs.docker.com/install/linux/docker-ce/ubuntu/)
- [Installing Percona Monitoring and Management (PMM) for the First Time](https://www.percona.com/blog/2017/02/24/installing-percona-monitoring-and-management-pmm-for-the-first-time-2/)
- [Deploying Percona Monitoring and Management](https://www.percona.com/doc/percona-monitoring-and-management/deploy/index.html#deploy-pmm-client-server-connecting)

## Install the integrated PMM server using Docker

### Install Docker

1. Uninstall old versions:

```bash
$ sudo apt-get remove docker docker-engine docker.io containerd runc
```

2. Update the apt package index:

```bash
$ sudo apt-get update
```

3. Install packages to allow apt to use a repository over HTTPS:

```bash
$ sudo apt-get install \
    apt-transport-https \
    ca-certificates \
    curl \
    gnupg-agent \
    software-properties-common
```

4. Add Docker’s official GPG key:

```bash
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
```

Verify that you now have the key with the fingerprint `9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88`, by searching for the last 8 characters of the fingerprint.

```bash
$ sudo apt-key fingerprint 0EBFCD88
pub   4096R/0EBFCD88 2017-02-22
      Key fingerprint = 9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
uid                  Docker Release (CE deb) <docker@docker.com>
sub   4096R/F273FCD8 2017-02-22
```

5. Use the following command to set up the stable repository:

```bash
$ sudo add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
```

6. Update the apt package index:

```bash
$ sudo apt-get update
```

7. Install the latest version of Docker CE and containerd:

```bash
$ sudo apt-get install docker-ce docker-ce-cli containerd.io
```

8. Verify that Docker CE is installed correctly by running the hello-world image:

```bash
$ sudo docker run hello-world
Unable to find image 'hello-world:latest' locally


latest: Pulling from library/hello-world
1b930d010525: Pulling fs layer
1b930d010525: Pull complete
Digest: sha256:xxxx...
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.
To generate this message, Docker took the following steps: 1. The Docker client contacted the Docker daemon. 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.    (amd64) 3. The Docker daemon created a new container from that image which runs the    executable that produces the output you are currently reading. 4. The Docker daemon streamed that output to the Docker client, which sent it    to your terminal.
To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/

```

### Setting Up a Docker Container for PMM Server

1. Pull the latest versions from Docker Hub:

```bash
$ sudo docker pull percona/pmm-server:latest
```

For version 1.x, use the below command:

```bash
$ sudo docker pull percona/pmm-server:1
```

2. Create a container for persistent PMM data:

```bash
$ sudo docker create \
   -v /opt/prometheus/data \
   -v /opt/consul-data \
   -v /var/lib/mysql \
   -v /var/lib/grafana \
   --name pmm-data \
   percona/pmm-server:latest /bin/true
```

```bash
$ sudo docker ps -a
CONTAINER ID        IMAGE                       COMMAND             CREATED             STATUS                     PORTS               NAMES
b26cb590b36c        percona/pmm-server:latest   "/bin/true"         11 seconds ago      Created                                        pmm-data
98025b2672e2        hello-world                 "/hello"            2 minutes ago       Exited (0) 2 minutes ago                       goofy_mcnulty
```

For version 1.x, use the below command:

```bash
$ sudo docker create \
   -v /opt/prometheus/data \
   -v /opt/consul-data \
   -v /var/lib/mysql \
   -v /var/lib/grafana \
   --name pmm-data \
   percona/pmm-server:1 /bin/true
```

```bash
$ sudo docker ps -a
CONTAINER ID        IMAGE                       COMMAND             CREATED             STATUS                     PORTS               NAMES
b26cb590b36c        percona/pmm-server:1   "/bin/true"              11 seconds ago      Created                                        pmm-data
98025b2672e2        hello-world                 "/hello"            2 minutes ago       Exited (0) 2 minutes ago                       goofy_mcnulty
```

The PMM server container is created.

3. Create and launch PMM Server in one command, use `docker run`:

```bash
$ sudo docker run -d \
   -p 80:80 \
   --volumes-from pmm-data \
   --name pmm-server \
   --restart always \
   percona/pmm-server:latest
```

```bash
$ sudo docker ps -a
CONTAINER ID        IMAGE                       COMMAND                CREATED             STATUS                     PORTS                         NAMES
26351b668b82        percona/pmm-server:latest   "/opt/entrypoint.sh"   5 seconds ago       Up 3 seconds               0.0.0.0:80->80/tcp, 443/tcp   pmm-server
b26cb590b36c        percona/pmm-server:latest   "/bin/true"            52 seconds ago      Created                                                  pmm-data
98025b2672e2        hello-world                 "/hello"               3 minutes ago       Exited (0) 3 minutes ago                                 goofy_mcnulty
```

For version 1.x, use the below command:

```bash
$ sudo docker run -d \
   -p 80:80 \
   --volumes-from pmm-data \
   --name pmm-server \
   --restart always \
   percona/pmm-server:1
```

```bash
$ sudo docker ps -a
CONTAINER ID        IMAGE                       COMMAND                CREATED             STATUS                     PORTS                         NAMES
26351b668b82        percona/pmm-server:1        "/opt/entrypoint.sh"   5 seconds ago       Up 3 seconds               0.0.0.0:80->80/tcp, 443/tcp   pmm-server
b26cb590b36c        percona/pmm-server:1        "/bin/true"            52 seconds ago      Created                                                  pmm-data
98025b2672e2        hello-world                 "/hello"               3 minutes ago       Exited (0) 3 minutes ago                                 goofy_mcnulty
```

The PMM server container is running. You can direct your browser to http://xxx.xxx.xxx.xxx/graph/ (IP address of your server). Since you have not added any monitoring services yet, the site will not show any data.

## Install the PMM client

1. Fetch the repository package:

```bash
$ wget https://repo.percona.com/apt/percona-release_latest.generic_all.deb
```

2. Install the downloaded repository package with `dpkg`:

```bash
$ sudo dpkg -i percona-release_latest.generic_all.deb
```

3. Update the apt package index:

```bash
$ sudo apt-get update
```

4. Install the PMM Client package:

```bash
$ sudo apt-get install pmm-client
```

## Connect the PMM client to the PMM Server

To connect a PMM Client, enter the IP address of the PMM Server as the value of the `--server` parameter to the `pmm-admin config` command:

```bash
$ sudo pmm-admin config --server 123.456.789.111
OK, PMM server is alive.

PMM Server      | 123.456.789.111
Client Name     | mijin
Client Address  | 123.456.789.112
```

## Start data collection services on the PMM client

To start [collecting data on each PMM Client connected to a PMM server](https://www.percona.com/doc/percona-monitoring-and-management/deploy/index.html#deploy-pmm-data-collecting), run the `pmm-admin add` command along with the name of the selected monitoring service.
For example, if you want to monitor MySQL server, the user is root, and the password is 1234, run the command as follows:

```bash
$ sudo pmm-admin add mysql --user root --password 1234 --query-source perfschema
[linux:metrics] OK, already monitoring this system.
[mysql:metrics] OK, now monitoring MySQL metrics using DSN root:***@unix(/tmp/mysql.sock)
[mysql:queries] OK, now monitoring MySQL queries from perfschema using DSN root:***@unix(/tmp/mysql.sock)
```

To see what is being monitored, run `pmm-admin list`:

```bash
$ sudo pmm-admin list
pmm-admin 1.17.0

PMM Server      | 123.456.789.111
Client Name     | mijin
Client Address  | 123.456.789.112
Service Manager | linux-systemd

-------------- ------ ----------- -------- ------------------------------- ---------------------------------------------
SERVICE TYPE   NAME   LOCAL PORT  RUNNING  DATA SOURCE                     OPTIONS                                      
-------------- ------ ----------- -------- ------------------------------- ---------------------------------------------
mysql:queries  mijin  -           YES      root:***@unix(/tmp/mysql.sock)  query_source=perfschema, query_examples=true
linux:metrics  mijin  42000       YES      -                                                                            
mysql:metrics  mijin  42002       YES      root:***@unix(/tmp/mysql.sock)
```

## Obtaining diagnostics data for support

You can retrieve collected data from your PMM Server in a single zip archive using this URL:

```bash
https://<address-of-your-pmm-server>/managed/logs.zip
```
## Stopping monitoring services

You can stop all services managed by PMM client using `pmm-admin stop`:

```bash
$ sudo pmm-admin stop --all
```

You can specify a monitoring service alias that you want to stop. To see which services are available, run `pmm-admin list`.

For example, to stop all services related to MySQL:

```bash
$ sudo pmm-admin stop mysql
```

You can also stop services of the PMM server using below command:

```bash
$ sudo docker stop pmm-server
```
