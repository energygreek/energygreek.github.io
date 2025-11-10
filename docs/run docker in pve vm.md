---
comments: true
date: 2025-02-07
tags: [pve,docker,lxc]
---
# difference of lxc container and docker container
* creating a lxc container will alloc a new partition from lvm(a new logic partition), create a root-fs and copy image into it, and the image is a zip file. 
* creating a docker container will mount docker image layers one by one into the host's fs

## best way
best way of running docker in pve is probably install docker inner a lxc container


## speed up docker installation
https://mirrors.tuna.tsinghua.edu.cn/help/docker-ce/

### remove old
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do apt-get remove $pkg; done

### setup repository
apt-get update
apt-get install ca-certificates curl gnupg
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://mirrors.tuna.tsinghua.edu.cn/docker-ce/linux/debian \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  tee /etc/apt/sources.list.d/docker.list > /dev/null

### install 
apt-get update
apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin


## copy image to another host

docker save <image> | bzip2 | ssh user@host docker load


## fix problem 'modprobe: FATAL: Module fuse not found in directory /lib/modules/6.1.10-1-pve'

enable the Options of your container (Select the container on the left-hand side > Options > Features) and then enable FUSE

***warn*** Just be aware that FUSE in LXC containers can cause some problems (no snapshots, no suspend backup, lower security since there is less containment).

## convert docker image to lxc image
https://www.reddit.com/r/Proxmox/comments/u9ru5c/comment/i640ixf/?utm_source=share&utm_medium=web3x&utm_name=web3xcss&utm_term=1&utm_content=share_button

https://gist.github.com/flaki/9a6c58447e50219a9b77a7cd15433896