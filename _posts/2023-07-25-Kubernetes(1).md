---
title: Kubernetes 실습 환경 구축(1)
author: SangkiHan
date: 2023-07-25 13:39:00 +0900
categories: [Kubernetes]
tags: [Kubernetes]
---
------------

## Kubernetes란?


## Kubernets 설치법

### master node
``` bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /usr/share/keyrings/kubernetes-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

``` bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

```오류발생시```
![Kubernetes](/assets/img/post/2023-07-25-Kubernetes(1)/1.PNG)

``` bash
sudo rm /etc/containerd/config.toml
sudo systemctl restart containerd
rm -rf /var/lib/etcd
sudo kubeadm init
```

kubeadm join 이하는 worker node와 master node를 연결할 때 필요하기 때문에 미리 복사해두자

### workder node
``` bash
sudo apt update
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /usr/share/keyrings/kubernetes-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update
sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```


``` bash
master node에서 저장한 kubeadm join 입력
```

### 확인
``` bash
kubectl get nodes
```