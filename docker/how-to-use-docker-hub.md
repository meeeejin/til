# How to use Docker Hub

## Reference

- [Docker Docs: Repositories](https://docs.docker.com/docker-hub/repos/)

## Creating repositories

Docker Hub에 로그인하고 `Create Repository`를 클릭해 새로운 repository를 생성한다:

![create-repo](https://docs.docker.com/docker-hub/images/repos-create.png)

## Pushing a Docker container image to Docker Hub

이미지를 Docker Hub로 push 하려면, Docker Hub의 username과 repository 이름을 사용해 로컬 이미지의 이름을 지정해야한다. 다음 방법 중 하나를 사용하면 된다:

- 빌드할 때, `docker build -t <hub-user>/<repo-name>[:<tag>]`를 통해 이름 지정

- `docker tag <existing-image> <hub-user>/<repo-name>[:<tag>]`로 기존 로컬 이미지에 태그를 다시 생성해 이름 지정

- `docker commit <existing-container> <hub-user>/<repo-name>[:<tag>]`로 변경 사항을 commit할 때 이름 지정

로컬 이미지의 이름을 생성한 후, 아래 명령어를 사용해 push 할 수 있다:

```bash
$ docker push <hub-user>/<repo-name>:<tag>
```

위 과정을 종합해 예를 들자면, 아래와 같이 로컬 이미지(`test`)를 Docker Hub repository에 push 할 수 있다. 이 때, Docker Hub의 username은 `meeeejin`이고, repository 이름은 `lb-mysql`, tag는 `test`다:

```bash
$ sudo docker login
$ sudo docker commit test meeeejin/lb-mysql[:test]
$ sudo docker push meeeejin/lb-mysql[:test]
```

`tag`를 따로 명시하지 않으면, 자동으로 `latest` tag가 붙는다.