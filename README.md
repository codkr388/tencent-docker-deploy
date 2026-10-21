# 腾讯云docker部署：新手从选机型到容器跑通，含镜像加速配置与拉取超时处理

在腾讯云上完成 Docker 部署，最短的路径是三步：买一台 64 位 Linux 云服务器，用腾讯云预置的软件源装 Docker，再配好镜像加速地址把容器拉起来。整个流程本身不长，卡住新手的地方高度集中在两处——选错机器类型，以及 `docker pull` 拉到一半超时。

**选购速览**：只想先跑通一个容器，买一台入门配置的 Linux 云服务器就够了，Ubuntu 22.04 或 TencentOS Server 4 都可以，按量计费试完随时释放；完全不想碰命令行，就选带 Docker 基础镜像的轻量应用服务器，在控制台点几下建容器。具体金额在购买页选完配置才会显示，地域和机型一变价格就变，这里不列固定数字。

👉 [查看腾讯云云服务器当前的配置与计费方式](https://bit.ly/yunhuodong)

## 先选对机器：三种在腾讯云跑 Docker 的方式

这三种方式的操作难度差得很远。云服务器 CVM 给你完整 root 权限，装什么、开什么端口全部自己定；轻量应用服务器把「选 Docker 镜像 → 填容器配置」做成了控制台表单；CloudBase 云托管更省事，你只要提供一个镜像地址，服务器根本不用碰。

| 部署方式 | 适合谁 | 操作方式 | 计费方式 | 入口 |
| --- | --- | --- | --- | --- |
| 云服务器 CVM | 需要自定义环境、跑多个服务、要 GPU 或特殊内核 | SSH 登录后按系统命令安装 Docker，自己管容器 | 按实际使用的资源计费，配置与地域影响报价 | [ 选一台 CVM 开始](https://bit.ly/yunhuodong) |
| 轻量应用服务器（Docker 基础镜像） | 跑一两个容器的小项目，想少敲命令 | 控制台选 Docker 基础镜像，填镜像地址、端口、环境变量建容器 | 以下单页显示的套餐为准 | [ 查看轻量应用服务器](https://bit.ly/yunhuodong) |
| CloudBase 云托管 | 已有镜像，只想把服务发布出去 | 填镜像地址 + 服务名 + 端口，创建后拿到默认访问域名 | 以下单页显示为准 | [ 查看云托管方案](https://bit.ly/yunhuodong) |

这三条路径可以组合使用。常见的组合是「CVM 装 Docker + TCR 存私有镜像」，或者「本地构建好镜像推到 TCR，再在云托管上跑」。

## 在 CVM 上装 Docker：按你的系统选命令

腾讯云公共镜像里已经预置了 docker 源，不需要去 Docker 官网照抄那套源配置，直接用系统包管理器装就行。前提只有一个：64 位系统，内核版本至少 3.10，腾讯云现在主流的公共镜像都满足。

👉 [先挑一台适合跑 Docker 的云服务器](https://bit.ly/yunhuodong)

### TencentOS Server 4 / OpenCloudOS 9.0

这两个版本的公共镜像预置了腾讯云的 docker 源，两条命令解决：

bash
sudo yum install docker -y
sudo systemctl start docker


装完执行 `sudo docker info`，能打出一屏信息就说明成功了。

### TencentOS Server 3

预置的是 docker-ce 源，用 dnf：

bash
sudo dnf install -y docker-ce --nobest
sudo systemctl start docker


### CentOS 8.2 / OpenCloudOS 8.0 / CentOS 7.9

这几个版本要手动把 Docker 官方源替换成腾讯云镜像，否则安装时会去连 download.docker.com。CentOS 8.2 和 OpenCloudOS 8.0 用 dnf：

bash
sudo dnf config-manager --add-repo=https://mirrors.cloud.tencent.com/docker-ce/linux/centos/docker-ce.repo
sudo sed -i "s/download.docker.com/mirrors.tencentyun.com\/docker-ce/g" /etc/yum.repos.d/docker-ce.repo
sudo dnf install -y docker-ce --nobest
sudo systemctl start docker


CentOS 7.9 换成 yum：

bash
sudo yum-config-manager --add-repo=https://mirrors.cloud.tencent.com/docker-ce/linux/centos/docker-ce.repo
sudo sed -i "s/download.docker.com/mirrors.tencentyun.com\/docker-ce/g" /etc/yum.repos.d/docker-ce.repo
sudo yum install -y docker-ce
sudo systemctl start docker


### Ubuntu 22.04 / Debian 12.5

apt 系统要先过一遍 GPG key 和源列表。Ubuntu 的命令是：

bash
sudo apt-get update
sudo apt-get install ca-certificates curl -y
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://mirrors.cloud.tencent.com/docker-ce/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://mirrors.cloud.tencent.com/docker-ce/linux/ubuntu/ $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl start docker


Debian 12.5 把上面两处 `ubuntu` 换成 `debian` 即可。

### 如果是 TencentOS Server 2.4 (TK4)

这个镜像已经预置了 Docker，跳过安装步骤，直接 `sudo systemctl start docker` 就能用。挑系统的时候如果想省掉装 Docker 这一步，可以优先考虑它。

装完之后建议把当前用户加进 docker 组，省掉每条命令前面敲 sudo：

bash
sudo usermod -aG docker $USER
newgrp docker


## 装完先配镜像加速，不然 docker pull 很容易卡住

`docker pull` 报 `Client.Timeout exceeded while awaiting headers`，这不是 Docker 装坏了，而是从 Docker Hub 拉镜像的网络走不通。解决办法是在 `daemon.json` 里指定腾讯云的镜像加速地址。

加速地址在腾讯云容器镜像服务控制台获取，拿到后编辑 `/etc/docker/daemon.json`：

json
{
  "registry-mirrors": ["https://<你的加速地址>.mirror.tencentyun.com"]
}


保存后重启 Docker：

bash
sudo systemctl restart docker


再用 `docker pull nginx` 验证一次。这一步做完，后面拉公共镜像都会走腾讯云缓存。配好之后跑容器：

bash
docker run -d -p 80:80 --name mynginx nginx


`-d` 让容器在后台跑，`-p 80:80` 把容器 80 端口映射到主机 80 端口，`--name` 起个名字方便后续 `docker stop mynginx` 或 `docker exec -it mynginx bash` 进容器。安全组记得放行对应端口，不开的话浏览器访问不到。

👉 [开通云服务器，跟着上面的命令跑一遍](https://bit.ly/yunhuodong)

## 不想敲命令：轻量应用服务器上点几下就有容器

如果你的需求只是「跑一个容器」，完全不用碰 SSH。购买实例时选 Docker 基础镜像，然后在创建实例的流程里添加容器：填容器名称、从镜像仓库选镜像、选镜像版本（不填默认 latest）、需要的话补上启动命令和参数。

镜像仓库建议选和实例同地域的，跨地域访问会消耗公网流量，还受公网带宽影响。

高级设置里这几项值得展开说：

- **绑定端口**：填服务器端口和容器端口，协议支持 TCP、UDP、SCTP。服务器端口只填数字（比如 80），外部就能访问；填 `127.0.0.1:80` 则只有本机能访问。
- **环境变量**：对应 `docker run` 的 `-e`，可以覆盖镜像 Dockerfile 里定义的值。
- **挂载存储卷**：对应 `-v`，把宿主机目录挂进容器。填的服务器目录不存在会自动创建。
- **重启策略**：不重启 / 非正常退出时重启（默认 1 次，可指定最大次数）/ 总是重启。

实例已经买好但不是 Docker 镜像？可以在「容器管理」页签直接新建容器，不过这种实例得先重装系统才能用控制台的容器管理功能。

有个容易忽略的地方：控制台的「重建容器」相当于删掉旧容器、用新配置重新创建，容器内的非持久化数据会丢失，还会有一段服务中断。所以一开始就把需要保留的东西——数据库文件、上传目录、配置文件——通过挂载存储卷放到宿主机上。重建之后这些数据还在，省得返工。

## 镜像要有地方放：TCR 与云托管

自己构建的镜像总得有个仓库。腾讯云容器镜像服务 TCR 支持私有和公有镜像仓库，个人实例免费，不收取费用。流程是建命名空间、建镜像仓库，本地 `docker login ccr.ccs.tencentyun.com`（用户名填腾讯云账号 ID），给镜像打标签后 push 上去。

CloudBase 云托管走的是另一条路：拿现成镜像直接发服务。进云托管「从容器部署」，填镜像地址、服务名称、端口，创建完在概述页拿到默认访问域名就能测。限制是它基于 X86 架构运行，ARM 类型的镜像跑不了，推之前先确认架构。

云托管上还有一批端口都配好的开源项目能一键部署：n8n 用 5678，code-server 和 Stirling-pdf 用 8080，Nextcloud、Excalidraw、Crawl4ai 也在列表里。想快速起个自用服务，比在 CVM 上从头装省事不少。

## 几个真的会卡住人的坑

最常见的一批失败原因集中在这几处，装之前扫一眼能省不少时间。

- **Windows 云服务器跑不了 Linux Docker。** 普通 Windows 云服务器（包括轻量应用服务器）不支持二次虚拟化：WSL 1 受系统内核完整性限制跑不了 Linux Docker，WSL 2 需要硬件支持二次虚拟化也不行。要跑 Linux 容器就选 Linux 镜像。Windows 裸金属物理机想跑 Windows 容器，建议选 Server 2022。另外普通 Windows 云服务器也不支持安装 Docker Desktop。
- **32 位系统装不上。** Docker 要求 64 位系统且内核版本至少 3.10。
- **ARM 镜像在云托管上起不来。** 云托管基于 X86 架构，只支持 AMD 类型镜像。
- **`docker pull` 超时基本是源的问题。** 配好加速地址再试，不用急着卸载重装。
- **端口通了不代表外网能访问。** 安全组默认没开 80、443 这些端口，要去控制台放行。2375、2376 是 Docker 守护进程端口，除非确实需要远程管理，不要长期对公网开放。

## 常见问题

**腾讯云服务器能装 Docker Desktop 吗？**
普通 Windows 云服务器不行。腾讯云明确说明不支持在 Windows 普通云服务器（含轻量应用服务器）上安装 Docker Desktop，WSL 方案也受限于二次虚拟化。

**装 Docker 要选多大的配置？**
Docker 本身占用很小，跑一两个轻量容器，入门配置就够。真正吃资源的是容器里的应用，选配置时按应用来算，不用把 Docker 的开销算进去。

**一定要用腾讯云的镜像源吗？**
看你机器的网络情况。`docker pull` 默认走 Docker Hub，国内网络下经常超时；腾讯云公共镜像预置了自家源，配加速也顺路，用它能省掉不少等待。

**轻量应用服务器能自己装 Docker 吗？**
可以，但买实例时直接选 Docker 基础镜像更省事，之后在控制台管容器。注意如果实例当初不是用 Docker 镜像创建的，需要先重装系统才能用容器管理。

## 最后怎么选

- 想完全掌控环境、要跑多个服务或需要 GPU：CVM 装 Docker，系统选 Ubuntu 22.04 或 TencentOS Server 4，装完第一件事配镜像加速。
- 只想跑一两个容器、不想维护系统：轻量应用服务器 + Docker 基础镜像，控制台建容器，先把要持久化的目录挂出来。
- 已有镜像、只想发布服务：CloudBase 云托管，上传前确认镜像是 X86 架构。
- 需要私有镜像仓库：TCR，个人实例免费。

下单前多花两分钟确认两件事：地域选在离用户近的地方，拉镜像和外部访问都快；安全组只放行真正要用的端口，别一次全开。

👉 [按业务规模选好配置，从腾讯云控制台开通](https://bit.ly/yunhuodong)
