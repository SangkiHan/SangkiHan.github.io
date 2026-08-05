---
layout: post
title: "Jenkins 배포가 갑자기 실패한 이유: snap docker가 소켓을 가로챈 사건"
date: 2026-08-05 20:00:00 +0900
categories: [DevOps, Docker]
tags: [Docker, Jenkins, Linux, systemd, Troubleshooting]
---

personal-assistant를 배포하는 Jenkins 파이프라인이 갑자기 실패했다. 원인을 하나씩 쫓아가다 보니 겉으로 보인 에러들은 전부 곁가지였고, 진짜 문제는 몇 주 전에 실수로 설치된 snap docker가 도커 소켓을 조용히 가로채고 있었다는 것이었다. 삽질한 순서 그대로 정리한다.

## 문제 상황

`personal-assistant` 배포가 이 에러로 실패했다.

```
network log-agent_default declared as external, but could not be found
```

원인을 찾으려고 같이 떠야 하는 `log-agent-backend` 스택을 재배포했더니, 이번엔 다른 에러가 났다.

```
Error response from daemon: error while creating mount source path
'/var/lib/jenkins/chromadb': mkdir /var/lib/jenkins: read-only file system
```

서로 다른 에러라 처음엔 완전히 별개의 문제라고 생각했다.

## 삽질 1: 외부 네트워크가 없다?

`docker-compose.yml`은 ChromaDB에 직접 접근하려고 `log-agent_default`라는 **외부(external) 네트워크**에 참여하도록 되어 있다.

```yaml
networks:
  log-agent-net:
    name: log-agent_default
    external: true
```

`external: true`는 이미 존재하는 네트워크를 참조만 하겠다는 뜻이라, 그 네트워크가 없으면 바로 이 에러가 난다. 그래서 서버가 이상한 상태인가 싶어 기본적인 것부터 확인했다.

```bash
mount | grep " / "
# /dev/sda2 on / type ext4 (rw,relatime,errors=remount-ro)

df -h
# /dev/sda2  117G  63G  48G  57%  /

dmesg -T | tail -50
# (I/O 에러 없음, 평범한 docker network veth 로그뿐)
```

디스크도 멀쩡하고 마운트도 `rw`. 이상할 게 없었다.

## 삽질 2: read-only filesystem?

두 번째 에러는 진짜 read-only filesystem처럼 보였다. 그런데 확인해보니 권한도, 속성도 다 정상이었다.

```bash
findmnt /var/lib/jenkins        # 결과 없음 → 별도 마운트 아님
ls -ld /var/lib/jenkins
# drwxr-xr-x 31 jenkins jenkins 4096 8월 5 14:41 /var/lib/jenkins

lsattr -d /var/lib/jenkins
# --------------e------- (immutable 아님)

touch /var/lib/jenkins/_test && echo OK
# OK
```

호스트에서 직접 `touch`는 성공하는데 dockerd가 같은 경로에 `mkdir`을 할 때만 실패한다. 이 조합은 snap으로 설치된 Docker의 전형적인 증상이다. snap 컨파인먼트가 dockerd를 자기만의 마운트 네임스페이스에 가둬서, `/home`, `/mnt`, `/media` 같은 snap이 허용한 경로 밖은 호스트에서는 멀쩡히 쓰기 되는데 dockerd 입장에서는 read-only로 보인다.

```bash
which docker
# /usr/bin/docker

snap list | grep docker
# docker  29.6.1  3579  latest/stable  canonical✓  -
```

확실히 snap docker였다. 그런데 이건 진짜 원인이 아니라 증상 중 하나였을 뿐이다.

## 결정적 단서: `docker ps`가 텅 비어있다

서비스는 멀쩡히 응답하고 있는데 `docker ps`를 쳐도 컨테이너가 하나도 안 떴다. 이게 제일 이상했다. 실제로 포트를 누가 물고 있는지부터 찾았다.

```bash
sudo ss -tlnp | grep -E ':6002|:8002|:80 |:443'
```

```
LISTEN 0 511  0.0.0.0:443  users:(("nginx",...))
LISTEN 0 511  0.0.0.0:80   users:(("nginx",...))
LISTEN 0 4096 0.0.0.0:6002 users:(("docker-proxy",pid=939705,...))
LISTEN 0 4096    [::]:6002 users:(("docker-proxy",pid=939712,...))
```

`docker-proxy`가 살아서 personal-assistant 포트(6002)를 서빙하고 있다. 즉 컨테이너는 분명히 떠 있다. 그럼 `docker ps`는 왜 못 보는가. `dockerd` 프로세스를 확인했다.

```bash
ps aux | grep -i dockerd
```

```
root  3123318  /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
root  3870376  dockerd --group docker --exec-root=/run/snap.docker \
                --data-root=/var/snap/docker/common/var-lib-docker ...
```

**`dockerd`가 두 개 떠 있었다.** 하나는 5월 11일부터 쭉 돌던 apt 버전, 하나는 7월 24일부터 돌던 snap 버전. `/var/run/docker.sock`을 지금 누가 쥐고 있는지 확인했다.

```bash
sudo lsof /var/run/docker.sock
```

```
dockerd 3870376 root ... /var/run/docker.sock (snap 버전)
```

```bash
systemctl status docker.socket docker.service --no-pager
```

```
● docker.socket  Active: active (running) since Mon 2026-05-11 13:47:09
● docker.service Active: active (running) since Mon 2026-05-11 13:47:11
  Main PID: 3123318 (dockerd)
  CGroup:
    ├─939705 docker-proxy ... -host-port 6002 -container-port 8000
    ├─3123966 docker-proxy ... -host-port 9090 ...
    ├─3124149 docker-proxy ... -host-port 3000 ...
    ├─3125050 docker-proxy ... -host-port 9200 ...
```

apt로 설치한 원래 docker daemon(5월 11일부터)은 그대로 살아서 모든 실제 서비스를 물고 있었다. 그런데 `docker` CLI가 붙는 소켓 파일(`/var/run/docker.sock`)은 나중에 설치된 snap dockerd가 가지고 있었던 것이다. `ls -la` 결과 소켓 파일의 mtime도 7월 24일 17:52 — snap docker가 시작한 바로 그 시각과 정확히 일치했다.

## 근본 원인

타임라인으로 정리하면 이렇다.

1. **5월 11일**: apt로 Docker Engine 설치. 이후 personal-assistant, chromadb 등 모든 서비스가 이 daemon 위에서 운영됨.
2. **7월 24일**: snap docker가 어쩌다 추가로 설치됨. snap docker도 기본적으로 `/var/run/docker.sock`을 쓰기 때문에, 이 시점부터 그 경로가 snap daemon 소유로 바뀜.
3. 그 이후로 사람이 치는 `docker` 명령도, Jenkins 파이프라인의 `docker build`/`docker compose`도 **전부 컨테이너가 하나도 없는 텅 빈 snap daemon**에게 갔다. apt daemon과 거기서 돌던 실제 서비스는 그대로 살아있었지만 CLI로는 더 이상 보이지 않게 됐다.
4. 처음 봤던 "network not found"와 "read-only filesystem" 에러는 둘 다 이 유령 daemon이 만든 증상이었다. 네트워크는 snap daemon 입장에서 진짜로 존재한 적이 없었고, snap 컨파인먼트 때문에 디렉토리 생성도 막혔던 것.
5. 두 달 넘게 아무 일도 없었던 이유는, 그동안 배포가 기존 컨테이너를 그대로 재사용했거나 실패 없이 지나갔기 때문이다. 이번에 `docker compose down` 후 네트워크·컨테이너를 처음부터 새로 만들어야 하는 상황이 되면서, snap daemon의 빈 상태와 컨파인먼트 문제가 동시에 겉으로 드러난 것이다.

## 해결

원본 apt daemon은 건드릴 게 없었다. 나중에 잘못 끼어든 snap docker만 정리하면 된다.

```bash
sudo systemctl stop snap.docker.dockerd.service
sudo snap remove docker --purge
sudo systemctl restart docker.socket docker.service
```

`docker.service`를 재시작해도 실제 컨테이너는 끊기지 않는다. dockerd는 `--containerd=/run/containerd/containerd.sock`으로 별도 프로세스인 containerd에 컨테이너 관리를 위임하기 때문에, dockerd 자체가 재시작돼도 이미 떠 있는 컨테이너는 영향을 받지 않는다.

정리 후 `docker ps`를 다시 치니 그동안 안 보이던 personal-assistant, chromadb를 포함한 모든 컨테이너가 정상적으로 나타났다. 이후 Jenkins 배포도 문제없이 통과했다.

## 회고

가장 결정적인 단서는 "서비스는 살아있는데 `docker ps`가 비어있다"는 모순이었다. 앞의 두 에러(네트워크 없음, read-only filesystem)만 붙잡고 있었다면 계속 엉뚱한 곳을 팠을 것이다. `docker ps`와 실제 서비스 상태가 어긋나면, 설정이나 권한보다 먼저 **`dockerd` 프로세스 개수와 `/var/run/docker.sock`을 누가 쥐고 있는지**부터 확인하는 게 맞다.

```bash
ps aux | grep -i dockerd
sudo lsof /var/run/docker.sock
```

재발 방지를 위해 이 서버에는 snap docker를 다시 설치하지 않도록 해뒀다. apt와 snap 두 종류의 docker가 같은 호스트에 공존하면 소켓 경로가 겹쳐서 언제든 같은 문제가 재발할 수 있다.

---

➡️ 시리즈 인덱스: [Slack으로 부리는 개인 비서](/posts/personal-assistant-intro)
