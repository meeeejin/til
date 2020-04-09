# Container networking

## Reference

- [Docker Docs: Container networking](https://docs.docker.com/config/containers/container-networking/#dns-services)
- [Docker Docs: docker port](https://docs.docker.com/engine/reference/commandline/port/)

## Published ports

컨테이너를 만들 때, default로는 컨테이너의 포트가 외부에 게시되지 않는다. Docker 외부 서비스 또는 컨테이너의 네트워크에 연결되지 않은 Docker 컨테이너에서 포트를 사용할 수 있게 하려면 `--publish` 또는 `-p` 플래그를 사용해야 한다. `-p [호스트 포트]:[Docker 컨테이너 포트]`의 형식으로 사용한다:

| Flag value | Description |
| :--------: | :---------: |
| `-p 8080:80` | 컨테이너의 TCP 포트 80을 Docker 호스트의 포트 8080에 맵핑 |
| `-p 192.168.1.100:8080:80` | 컨테이너의 TCP 포트 80을 Docker 호스트의 IP 192.168.1.100에 맵핑|
| `-p 8080:80/udp` | 컨테이너의 UDP 포트 80을 Docker 호스트의 포트 8080에 맵핑 |
| `-p 8080:80/tcp -p 8080:80/udp` | 컨테이너의 TCP 포트 80을 Docker 호스트의 TCP 포트 8080에 맵핑하고, 컨테이너의 UDP 포트 80을 Docker 호스트의 UDP 포트 8080에 맵핑 |

## docker port

`docker port`를 사용해 컨테이너에 대한 포트 맵핑 정보를 조회할 수 있다. 아래의 명령어를 사용해 조회한다:

```bash
docker port CONTAINER [PRIVATE_PORT[/PROTO]]
```

`PRIVATE_PORT`를 명시하지 않으면 전체 맵핑 정보를 조회할 수 있고, `PRIVATE_PORT`를 명시하면 해당 포트에 맵핑된 특정 포트를 찾을 수 있다:

```bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                                            NAMES
b650456536c7        busybox:latest      top                 54 minutes ago      Up 54 minutes       0.0.0.0:1234->9876/tcp, 0.0.0.0:4321->7890/tcp   test
$ docker port test
7890/tcp -> 0.0.0.0:4321
9876/tcp -> 0.0.0.0:1234
$ docker port test 7890/tcp
0.0.0.0:4321
$ docker port test 7890/udp
2014/06/24 11:53:36 Error: No public port '7890/udp' published for test
$ docker port test 7890
0.0.0.0:4321
```