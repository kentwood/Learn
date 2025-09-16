## 题目
This question needs to be solved on node node01. To access the node using SSH, use the credentials below:

username: bob
password: caleston123

As an administrator, you need to prepare node01 to install kubernetes. One of the steps is installing a container runtime. Install the cri-docker_0.3.16.3-0.debian.deb package located in /root and ensure that the cri-docker service is running and enabled to start on boot.

## 解题
1. 先登录到node01
```bash
ssh bob@node01
```
2. 切换到root用户
```bash
sudo su -
```
3. 使用apt安装cri-docker
```bash
apt install /root/cri-docker_0.3.16.3-0.debian.deb
```
4. 启动cri-docker
```bash
systemctl start cri-docker
```
5. 设置cri-docker为自启动
```bash
systemctl enable cri-docker
```
