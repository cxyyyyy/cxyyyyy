# EAMic（ValueApex）Docker 部署项目知识库

> **用途**：本项目 EAMic 实施 / Docker 部署 / 运维支持的知识库。
> **来源文档**：
> - 《docker [ValueApex].pdf》（16 页，EAMic Docker 部署手册，官网 wiki: `https://valueapex.com/extranet/documentation/wiki/doku.php?id=docker`）
> - 《准备系统及强化安全_linux [ValueApex].pdf》（4 页，官网 wiki: `https://valueapex.com/extranet/documentation/wiki/doku.php?id=准备系统及强化安全_linux`）
> **整理日期**：2026-10-27
> **约定**：文中出现的具体密码、IP 均为部署文档中的示例/默认值，实际部署以项目交付资料为准。

---

## 0. 部署概览（一句话）

EAMic 仅支持 **Linux + Docker** 部署；**应用服务器**运行 EAMic 容器（标准镜像 `images.easydev.fun/eamic-docker` 或定制镜像），**数据库服务器**安装 MariaDB；AI 相关能力统一部署在应用服务器（AP Server）上，**不单独部署大模型**，只调用大模型 API（如云端 DeepSeek API 或客户已本地部署的大模型 API），已移除原 ai-proxy 组件，向量嵌入库使用本地 SQLite。

---

## 1. 标准部署流程（7 步，按顺序执行）

| 步骤 | 动作 | 执行位置 |
|---|---|---|
| 1 | 准备系统及强化安全 | 应用服务器 + 数据库服务器 |
| 2 | 安装 Docker | 应用服务器 |
| 3 | 安装数据库 | 数据库服务器 |
| 4 | 上传 Docker 镜像到服务器 | 应用服务器 |
| 5 | eamic_storage 文件部署 | 应用服务器 |
| 6 | 启动/重启 eamic 容器 | 应用服务器 |
| 7 | 数据库初始化 | 应用服务器（容器内） |

部署后维护：更新镜像 → 自动更新数据库 → 查 docker 日志 → Troubleshooting 故障排查。

---

## 2. 系统准备及强化安全（两台服务器都要执行）

### 2.1 创建运维用户 eamic（禁用 root 远程登录）
> root 是危险用户，攻击者破解密码即获最高权限。需建立有 sudo 权限的新用户用于安装与运维，并向客户 IT/运维人员说明。

```bash
ssh root@<服务器IP>        # 尖括号内容替换为实际 IP，尖括号本身不需要
adduser eamic              # Redhat/CentOS 用: useradd eamic
passwd eamic               # 上一步已设密码则跳过
usermod -aG sudo eamic     # 加入 sudo 组
su - eamic                 # 切换到 eamic
```
建立 eamic 用户后，**之后所有操作都应使用 eamic 用户进行**。

### 2.2 系统初始化

**Debian/Ubuntu：**
```bash
sudo apt update
sudo apt install nano
sudo timedatectl set-timezone Asia/Shanghai
sudo apt install chrony
sudo systemctl start chronyd
sudo systemctl enable chrony
```

**Redhat/CentOS/RockyLinux/AlmaLinux：**
```bash
sudo yum update
sudo yum install nano
sudo timedatectl set-timezone Asia/Shanghai
sudo yum install ntp
sudo systemctl start ntp
sudo systemctl enable ntp
```

### 2.3 强化安全

**Debian/Ubuntu（ufw 防火墙 + 无人值守更新 + SSH + 日志轮转）：**
```bash
sudo systemctl status ufw          # 检查防火墙状态
sudo apt install ufw
sudo ufw status                    # 查看已开放端口
sudo ufw enable                    # 开启防火墙
sudo ufw allow ssh
sudo ufw allow https               # 443
sudo ufw allow http                # 80
sudo ufw allow 1280                # EAMic 端口
sudo ufw enable                    # 确保防火墙服务是打开状态
sudo apt install unattended-upgrades
sudo nano /etc/ssh/sshd_config     # 禁止 root 账号 ssh 登录
sudo systemctl reload sshd
sudo apt install logrotate
sudo nano /etc/logrotate.conf      # weekly 改成 monthly，4 改成 6（保留最近 6 个月日志）
```

**Redhat/CentOS/RockyLinux/AlmaLinux（firewalld + fail2ban + 自动更新 + SSH + 日志轮转）：**
```bash
sudo systemctl status firewalld
sudo firewall-cmd --list-all       # 列出所有开放端口
sudo systemctl start firewalld
sudo firewall-cmd --permanent --add-service=ssh
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --add-service=http   # 仅当客户接受 HTTP 时
sudo firewall-cmd --reload
sudo systemctl enable firewalld
# fail2ban（epel 源是装 fail2ban 和 phpmyadmin 的前置）
sudo yum install epel-release
sudo yum install fail2ban-firewalld
sudo nano /etc/fail2ban/jail_ssh.local   # 文件不存在就创建，写入：
# [DEFAULT] findtime = 5000
# [sshd] enabled = true maxretry = 1
sudo systemctl start fail2ban.service
sudo systemctl enable fail2ban.service
# 自动更新
sudo yum -y install yum-cron
sudo nano /etc/yum/yum-cron.conf    # download_updates=yes，apply_updates=yes
sudo systemctl start yum-cron
sudo systemctl enable yum-cron
# 禁用 root 远程登录
sudo nano /etc/ssh/sshd_config      # PermitRootLogin 设为 no（默认被 # 注释，需删 # 启用）
sudo systemctl reload sshd
# 日志轮转
sudo dnf install logrotate
sudo nano /etc/logrotate.conf       # weekly→monthly，4→6（保留 6 个月）
```

---

## 3. 操作系统建议与磁盘分区

### 3.1 操作系统
- 仅支持 Linux；**Windows Server 暂不支持**（需底层开虚拟化，较麻烦）。
- 推荐 **Debian 13**（长期支持版）。其他支持：Ubuntu 26.04 LTS、RedHat Enterprise Linux、Rocky Linux 10、Alma Linux 10（均建议 LTS 最新版）。
- **分类**：Debian 系列（Debian/Ubuntu，包管理 apt，防火墙 ufw）；Redhat 系列（Redhat/RockyLinux/AlmaLinux/EulerOS/KylinOS，包管理 dnf/yum，防火墙 firewalld）。
- 查看当前系统：`cat /etc/os-release`

### 3.2 磁盘分区建议
| 方案 | 适用 | 说明 |
|---|---|---|
| 1. 只有一个分区 | 大多数客户 | Debian 默认分区方式 |
| 2. 单独 /opt 分区且空间最大 | 有单独数据磁盘的客户 | 安装系统时仍建议方案1，之后加磁盘挂载到 /opt |
| **不建议**：单独 /home 分区且最大 | 个人电脑做法 | 不适合服务器 |

原因：EAMic 程序及文档附件放在 `/opt/eamic_storage`，占用空间大；单分区或 /opt 放在最大分区才能有效使用磁盘。

---

## 4. 安装 Docker【应用服务器】

### 4.1 Redhat/Rockylinux/AlmaLinux 系列（卸载并重装）
```bash
# 卸载旧版本
sudo yum remove docker docker-client docker-client-latest \
  docker-common docker-latest docker-latest-logrotate \
  docker-logrotate docker-engine
# 安装（阿里云源）
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
# 设置服务 + 启动 + 验证
sudo systemctl enable docker
sudo systemctl start docker
docker --version
# 参考: https://docs.docker.com/engine/install/centos/
```

### 4.2 Debian/Ubuntu 系统
```bash
# 删除旧版本
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done
# 设置软件源（阿里云镜像）
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
# Debian:
sudo curl -fsSL http://mirrors.aliyun.com/docker-ce/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] http://mirrors.aliyun.com/docker-ce/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
# Ubuntu: 将上一条中的 debian 分别替换为 ubuntu
sudo apt-get update
# 安装
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
# 启动 + 验证
sudo systemctl enable docker
sudo systemctl start docker
docker --version
# 参考: https://docs.docker.com/engine/install/debian/
```

### 4.3 把 eamic 加入 docker 组
```bash
sudo usermod -aG docker eamic   # 若当前 Linux 账号不是 eamic，替换为当前账号
# exit 退出当前登录，重新登录生效（免 sudo 执行 docker 命令不报 permission denied）
```

### 4.4 更新 Docker 引擎
- **Debian/Ubuntu**：`sudo apt update` → `sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin` → `sudo systemctl restart docker` → 验证 `docker --version`、`docker info`
- **Redhat RHEL/Rockylinux/Almalinux**：`sudo dnf makecache` → `sudo dnf update -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin` → `sudo systemctl restart docker` → 验证同上

---

## 5. 安装数据库【数据库服务器】—— MariaDB

```bash
# 安装
sudo apt install mariadb-server        # Debian
sudo yum install mariadb-server        # Redhat/Rockylinux/Almalinux
# 设置服务及启动
sudo systemctl enable mariadb
sudo systemctl start mariadb
sudo systemctl status mariadb          # 按 q 退出
# 初始化（设置密码）
sudo mariadb-secure-installation
```

### 5.1 创建 root@'%' 远程账号
```bash
mysql -u root -p
```
```sql
CREATE USER 'root'@'%' IDENTIFIED BY 'Matrixuall';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```
执行完输入 `exit` 退出数据库。

### 5.2 数据库配置（mysql 配置文件）
配置文件位置：**Debian** 通常 `/etc/mysql/mariadb.conf.d/50-server.cnf`；**RockyLinux/AlmaLinux** 通常 `/etc/my.cnf`。在 `[mysqld]` 段下配置：

```ini
bind-address = 0.0.0.0                  # 允许远程连接

# 打开 slow query log（慢查询日志）
slow_query_log = 1
slow_query_log_file = /var/log/mysql/mysql-slow.log   # 按实际调整路径
long_query_time = 120                   # 120 秒，即 2 分钟

# innodb_buffer_pool_size 是最关键参数之一，默认 128M 不适合生产，按内存调整，重启生效
innodb_buffer_pool_size=40              # 示例值，单位按实际内存调整
sql_mode="ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION"
```
验证 buffer pool：登录数据库后执行 `SELECT @@innodb_buffer_pool_size / 1024 / 1024 AS innodb_buffer_pool_size_MB;`

重启数据库：`sudo systemctl restart mariadb`

如需重置 root@% 密码：
```sql
SET PASSWORD FOR 'root'@'%' = PASSWORD('Matrixvall+');
```

### 5.3 防火墙放行 3306
- **RedHat/AlmaLinux/Rockylinux**：
```bash
sudo firewall-cmd --list-all
sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent
sudo firewall-cmd --reload
```
- **Debian/Ubuntu**：
```bash
sudo ufw status
sudo ufw allow 3306
```

### 5.4 备份数据库【重要】
```bash
# 1. 创建目录
mkdir -p /home/eamic/eamic_backup/{daily,monthly}
# 2. 上传备份脚本 backup_script_linux 到 /home/eamic/eamic_backup，赋予执行权限
chmod a+x /home/eamic/eamic_backup/backup_script_linux
# 3. 修改 backup_script_linux 中的 MYSQL_USER / MYSQL_PASSWORD / MYSQL_BACKUP_DIR / MYSQL_HOST
# 4. 设置 crontab 任务
sudo crontab -e     # 粘贴: 0 3 * * * /home/eamic/eamic_backup/backup_script_linux
# 5. 手动执行一次，确认有备份文件且大小合理
/home/eamic/eamic_backup/backup_script_linux
# 6. 第二天、第二周复查备份文件
# 7. 异地备份：把备份文件复制到其他服务器
```

---

## 6. 上传 Docker 镜像到服务器

**前提说明**：服务器能访问 `easydev.fun` 可直接在服务器上拉取；无外网则在本地电脑装 Docker Desktop（https://www.docker.com/products/docker-desktop/）下载后上传；若 Pierre 直接发镜像压缩包，无需解压，直接上传到服务器，并保证 compose.yaml 的 `image:` 与实际镜像名一致。

```bash
# (1) 查 docker 登录密码: Extranet | Documentation | Product | Linux - Docker | Docker registry password
# (2) 登录 docker 仓库（用户名 project）
docker login images.easydev.fun
# (3) 拉取镜像
docker pull images.easydev.fun/eamic-docker                          # 标准系统镜像
docker pull images.easydev.fun/项目编码-项目名称-名称小写              # 定制开发镜像，如: 22150-enox
# 注意: docker login 与 docker pull 是否加 sudo 必须保持一致，否则报 no basic auth credentials
# (4) 查看镜像
docker images
# (5) 更改 tag（仅当镜像名与 compose.yaml 的 image:xxx 不一致时才做，不要盲目执行）
sudo docker tag images.easydev.fun/eamic-docker eamic-docker         # 标准
sudo docker tag images.easydev.fun/24069-jinshiyuan 24069-jinshiyuan # 定制示例
# (6) 保存（若已在服务器上下载则跳过）
docker save -o eamic-docker.tar.gz eamic-docker
# (7) 上传到服务器 /home/eamic（无需解压），加载（若已在服务器上下载则跳过）
sudo docker load -i eamic-docker.tar.gz                             # 替换成实际文件名
```

**关于镜像的说明**：容器使用哪个镜像取决于 compose.yaml 里 `image` 的值，例如：
```yaml
services:
  eamic:
    image: images.easydev.fun/eamic-docker   # 示例1: 完整名称
    # image: eamic-docker                     # 示例2: 简短名称
```
更新镜像时需检查该名称与最新镜像是否一致，不一致则改名称或改 tag。

---

## 7. eamic_storage 文件部署

### 7.1 创建目录
```bash
df -h                          # 先查看磁盘分区，选空间最大的分区
mkdir /opt/eamic_storage       # 只有 / 根分区且空间足够时
mkdir /最大的分区目录/eamic_storage   # 多分区时；后续文档中的 /opt/eamic_storage 用实际目录代替
```
磁盘太小及时与客户沟通扩容。

### 7.2 下载并上传文件到 /opt/eamic_storage
| 文件 | 说明 |
|---|---|
| compose.yaml | 容器编排 |
| .env | 环境变量（注意前面有小数点，是隐藏文件） |
| v2、v2.js | **新版标准镜像（2026-04-01 以后）已包含，无需此文件**；定制开发除外 |
| config | 配置文件 |
| files | 附件等；多台应用服务器需先挂载 NAS，再映射到挂载目录（见 NAS 挂载文档） |

```bash
# 权限：files、config 允许写入（33 即 www-data 用户 ID）
sudo chown -R 33:33 /opt/eamic_storage/files
sudo chmod -R g+s /opt/eamic_storage/files
sudo chmod -R a+w /opt/eamic_storage/files
sudo setfacl -R -d -m g:33:rwx /opt/eamic_storage/files
sudo chmod -R a+w /opt/eamic_storage/config
```

**定制镜像**：打开 compose.yaml，把 `image: eamic-docker:latest` 改成实际镜像名（`docker images` 可查），例如 `image: 24069-jinshiyuan:latest`。
**新版标准镜像**：在 compose.yaml 中把 v2、v2.js 映射行用 `#` 注释掉（已包含在镜像中）。

### 7.3 eamic_db（phpMyAdmin）
- 下载 phpMyAdmin（https://www.phpmyadmin.net/），上传到 /opt/eamic_storage，解压后文件夹改名为 `eamic_db`。
- 把 `config.sample.inc.php` 改名为 `config.inc.php`。
- 更新 `$cfg['Servers'][$i]['host']` = 数据库 IP：单独数据库服务器 → 填数据库服务器 IP；同一台 → 填实际 IP。
- `$cfg['blowfish_secret']` 填任意 32 个字符。

### 7.4 修改配置文件
- 修改 `.env`、`config/config.inc.php` 中的数据库 IP（同上）。
- `config/config.inc.php` 中 `$cfg['in_plant'] = 'plant_name';`，把 `plant_name` 换成客户英文名（10 个字符）。

### 7.5 index.php 与 eamic_license
- 向 **Pierre** 索取首页 `index.php` 及 `eamic_license`。需向 Pierre 提供：客户**中文名称**、**英文名称**、**容器里的 IP 地址**（进入容器执行 `hostname -I`，通常是 172.20.0.5；**是容器 IP，不是宿主机 IP**）。
- 拿到后上传到 /opt/eamic_storage。
- 注意：定制开发且有 git 代码分支的项目，index.php 镜像里已有，不再需要；但 **eamic_license 依然需要**。

### 7.6 启动/重启容器
```bash
# 启动
cd /opt/eamic_storage
sudo docker compose up -d
# 停止
cd /opt/eamic_storage
sudo docker compose down
# 重启 = 先停止，再启动
```
> 报错 `error mounting ... to rootfs at ...`：通常是启动前 compose.yaml 指定的映射文件不存在（如 index.php、eamic_license），/opt/eamic_storage 下出现的是同名**文件夹**（ls -l 第一位是 d）而不是文件（第一位是 -）。解决：停止容器 → 删除文件夹 → 上传缺失的文件 → 启动容器。

---

## 8. 防火墙端口放行

通常防火墙未启用，端口可直接访问；若启用了防火墙，需放行 EAMic 端口（默认 1280）：
- **RedHat/AlmaLinux/Rockylinux**：
```bash
sudo firewall-cmd --list-all
sudo firewall-cmd --zone=public --add-port=1280/tcp --permanent
sudo firewall-cmd --reload
```
- **Debian（ufw）**：`sudo ufw allow 1280`

⚠️ 注意：Debian 用 ufw，Redhat 系列用 firewalld；**不要在 Debian 上装 firewalld，不要在 Redhat 上装 ufw**。此步没有防火墙就不需要安装。

---

## 9. 数据库初始化与登录

```bash
# 进入容器
sudo docker container ls                                     # 查看容器名
sudo docker exec -it eamic_storage-eamic-1 bash             # 容器名按上面查到的替换
# 创建并导入数据库（容器内执行）
php console app:install
```
- 系统自动创建并导入 **mic_admin、mic00001** 数据库，执行安装并显示 EAMic 登录账号密码。
- 提示输入 IP 地址时，输入**应用服务器（AP Server）即宿主机**的 IP（`hostname -I` 查询）。
- 浏览器访问 `http://<应用服务器IP>:<端口>`（端口在 compose.yaml 定义，通常 **1280 或 80**）登录 EAMic 系统。
- 访问数据库：`http://<ap_server_ip>:1280/eamic_db`

### 容器操作速查
```bash
sudo docker container ls                       # 查看容器名
sudo docker exec -it eamic_storage-eamic-1 bash # 连接容器
sudo docker cp 本地文件路径 容器名称:容器里的路径  # 上传文件到容器（示例: docker cp /home/eamic/register_get.php eamic_storage_eamic_1:/var/www/html/api/v1/assets）
```
⚠️ 上传文件到服务器/容器只是**临时排查**，容器重启后消失。程序文件需打包进镜像；前端文件通过 compose.yaml 映射进容器。

### 如何确定当前系统使用的镜像
- **标准系统镜像**：`images.easydev.fun/eamic-docker`
- **定制开发镜像**：`images.easydev.fun/项目编码-项目名称`，如 `images.easydev.fun/22150-enox`
- 老镜像域名 `dev.valueapex.com:5000` 已废弃，新拉取的镜像都用 `images.easydev.fun`。
- 查看方法：打开 /opt/eamic_storage/compose.yaml 查看 image 的值。

---

## 10. 日常维护

### 10.1 更新镜像
思路：**A.** 拉取(a1)或上传(a2)新镜像到应用服务器 → **B.** 确保 compose.yaml 的 image 值与新镜像一致 → **C.** 重启容器。
```bash
# a2.1 上传（如直接服务器拉取则跳过）：复制 eamic-docker.tar.gz 到 /home/eamic（无需解压）
# a2.2 加载
sudo docker load -i eamic-docker.tar.gz
# b. 核对 compose.yaml 的 image 值
# c. 重启容器
cd /opt/eamic_storage
sudo docker compose down
sudo docker compose up -d
```
**检查是否最新**：`docker images` 看 CREATED 时间；进容器查看 `build-logs.txt`（含镜像构建日期 + 最后一次代码提交 commit）：
```bash
sudo docker exec -it eamic_storage_eamic_1 bash
# 容器内 /var/www/html 下 ls 可见 build-logs.txt
cat build-logs.txt
```

### 10.2 更新前端文件
- 标准 eamic-docker 镜像**已包含 v2、v2_jf**，无需单独更新；新版（2026-04-01 后）标准镜像在 compose.yaml 中把 v2、v2_lf/v2_jf 映射行注释掉后重启容器。
- 定制开发：把开发提供的 v2、v2_jf、v2_sp 等前端文件上传到 /opt/eamic_storage，替换原文件。

### 10.3 自动更新数据库
- 每次启动容器会自动执行镜像里的数据更新，更新镜像时停止再启动容器即可，无需手动干预。
- 未生效时手动执行：
```bash
sudo docker exec -it eamic_storage_eamic_1 bash
php console migrate
# 部分老的镜像用: php ./api/src/Shared/Helper/MigrateDatabaseCli.php migrate
# 数据库更新文件位置: /var/www/html/migrations/
```
- 查询更新结果：`SELECT * FROM mic_log_database_update ORDER BY query_id DESC;`

### 10.4 v2、jq 等文件区别（文件映射对照）
| 文件 | 宿主路径 | 容器路径 | 用途 | 代码仓库 | 重点/记忆 |
|---|---|---|---|---|---|
| v2/jq | 无 | /var/www/html/v2/jq | 主数据 EAMic 代表文件 | eamic2 / eamic-docker | v2 里没有 jq；jq/pages 用 jQuery，主数据非 Angular，代码都在镜像里 |
| assets/(v2,v2_sp,v2_jf) | /opt/eamic_storage/(v2,v2_sp,v2_jf) | /var/www/html/assets/(v2,v2_sp,v2_jf) | 前端浏览器界面 | 有 jq | 前端是单独代码库，在宿主 storage 中，通常 zip 解压更新 |
| login page | /opt/eamic_storage/index.php | /var/www/html/public/index.php | 登录界面/版权页 | 无 | Pierre 提供（发客户中文名称后）；定制开发有 git 分支的镜像已含，别再要 |
| Config file | /opt/eamic_storage/config | /var/www/html/config | 配置文件 | 无 | — |
| Customizations | /opt/eamic_storage/xx_customized | /var/www/html/某目录/customized | 定制开发 | 无 | 适用于标准镜像客户长期维护；有专门代码分支的客户定制文件放镜像里 |

> 若 /opt/eamic_storage/v2/ 里也有 jq 文件，可以删除。

### 10.5 自定义计划任务
标准计划任务已集成在镜像内 `/etc/cron.d/root.cron`。自定义任务在宿主机 crontab 添加，例如：
```bash
0 0 * * 1 docker exec eamic_storage-eamic-1 php /var/www/html/xxx.php
```
可执行容器内任意目录下的文件（此处只是举例）。

### 10.6 客户更换了数据库需要修改什么
1. 修改 `/opt/eamic_storage/` 下这三个文件中的数据库 IP 为新库地址：`.env`（隐藏文件）、`config.inc.php`、`eamic_db/config.inc.php`；**重启容器**。
2. 新数据库中 `mic_admin.mic_host` 表，把数据库 IP 改为新库地址。
3. 检查数据库账号：`mic00001@'host_ip'` 是否为 `%` 或应用服务器 IP，不是则改。
4. 检查密码：`mic00001@` 的实际密码需与 `mic_admin.mic_site.site_password` 一致。

### 10.7 如何查 docker 日志
```bash
docker logs eamic_storage-eamic-1          # 查看日志
docker logs --tail 500 eamic_storage-eamic-1  # 最后 500 行
docker logs -f eamic_storage-eamic-1       # 实时查看（ctrl-c 结束）
```

### 10.8 删除旧的镜像和容器（节省磁盘空间）
```bash
docker images                             # 列出所有镜像
docker image rm 镜像ID                    # 删除某个镜像
docker image prune                        # 删除所有没在用的镜像（先确保容器是启动状态）
docker ps -a                              # 列出所有容器（含未运行）; 或 docker container ls -a
docker container rm 容器ID                # 删除某个容器
docker container prune                    # 删除所有无用的容器
```

### 10.9 回退旧的镜像（Rollback）
新镜像有严重问题且短时间无法修复时回退：
```bash
docker images                # (1) 查看服务器上的镜像，示例: 24069-jinshiyuan latest d424b94b24c6 3 hours ago
cd /opt/eamic_storage        # (2) 停止容器
sudo docker compose down
docker tag 镜像ID 镜像标签    # (3) 改 tag，把镜像ID 与系统使用的镜像标签对应，如: docker tag 19756057e67e 24069-jinshiyuan
cd /opt/eamic_storage        # (4) 重启容器
sudo docker compose up -d
```

### 10.10 更改端口
EAMic 容器默认 1280 端口，改端口（以 80 为例）：
1. 打开 compose.yaml，把 `ports:` 下的 `- "1280:80"` 改成 `- "80:80"`
2. 重启容器：`sudo docker compose down` → `sudo docker compose up -d`

---

## 11. Troubleshooting 故障排查

### (1) PHP 错误日志
- EAMic 系统内：管理日志 - PHP 错误日志。
- 服务器宿主机路径：`/opt/eamic_storage/files/logs/php_logs/php_errors.log`
```bash
tail -n 100 /opt/eamic_storage/files/logs/php_logs/php_errors.log   # 最近 100 条
tail -f /opt/eamic_storage/files/logs/php_logs/php_errors.log       # 实时
```
- 日志目录不存在时创建并授权：
```bash
sudo mkdir -p /opt/eamic_storage/files/logs/php_logs
sudo chown -R 33:33 /opt/eamic_storage/files/logs
sudo chmod -R g+ws /opt/eamic_storage/files/logs
sudo chmod -R a+w /opt/eamic_storage/files/logs
sudo setfacl -R -d -m g:33:rwx /opt/eamic_storage/files/logs
```
- 容器内检查 `/usr/local/etc/php/conf.d/00-local.prod.ini`（老镜像是 `local.prod.ini`）中：
```ini
log_errors = On
error_log = /var/www/html/files/logs/php_logs/php_errors.log
```
不存在则添加，然后 `/etc/init.d/apache2 reload`

### (2) docker logs
见 10.7 如何查 docker 日志。

### (3) network - response 中显示错误日志（display_errors）
- **不适用场景**：浏览器报 `net::ERR_` 开头错误（如 net::ERR_CONNECTION_RESET）→ 请求还没到达应用服务器就被卡住，属于**网络层问题**，常见原因是客户网络屏蔽了 PUT、DELETE、PATCH、OPTIONS、TRACE、CONNECT 等 HTTP 方法 → 确认请求方法后联系客户 IT。
- **适用场景**：请求返回码 500 但响应内容为空 → 错误发生在底层，EAMic 捕获不到 → 临时开启 display_errors：
```bash
# 进入容器
cd /usr/local/etc/php/conf.d/
nano 00-local.prod.ini     # 老的镜像是 nano local.prod.ini
# 设置 display_errors = On
/etc/init.d/apache2 reload
```
- 重新访问有问题的页面，浏览器 network - response 中即显示 PHP 错误信息。
- ⚠️ 排查后出于安全考虑，**必须关闭**（`display_errors = off` + reload）。

### (4) 打开配置文件 debug 选项
- **适用场景**：network - response 显示 `'API ERROR - 请检查EAMic错误日志'`，但 EAMic 错误日志里没有新报错时。
- **方法**：容器内打开 `/var/www/html/config/config.inc.php`，设置 `$cfg['debug'] = true;`（没有就添加），重新访问页面，在 network - response 中查看具体报错。
- ⚠️ 排查后关闭：`$cfg['debug'] = false;`（开启 debug 会暴露报错细节，攻击者可能据此调整攻击路径）。

---

## 12. 关键信息速查

| 项目 | 值 |
|---|---|
| 镜像仓库 | images.easydev.fun（用户名 project，密码在 Extranet → Documentation → Product → Linux - Docker → Docker registry password 查询） |
| 标准镜像 | images.easydev.fun/eamic-docker |
| 定制镜像命名 | images.easydev.fun/项目编码-项目名称（如 22150-enox、24069-jinshiyuan） |
| 部署目录 | /opt/eamic_storage（多分区时取最大分区） |
| 容器默认端口 | 1280（可改为 80 等） |
| 数据库 | MariaDB，端口 3306，库：mic_admin、mic00001 |
| 数据库配置路径 | Debian: /etc/mysql/mariadb.conf.d/50-server.cnf；Rocky/Alma: /etc/my.cnf |
| 备份 | /home/eamic/eamic_backup（daily/monthly），cron: 0 3 * * * |
| 日志 | 容器: docker logs；PHP: /opt/eamic_storage/files/logs/php_logs/php_errors.log |
| 前端文件 | v2、v2_sp、v2_jf 等（新版标准镜像已内置，2026-04-01 后） |
| 联系人 | Pierre（index.php、eamic_license、定制镜像压缩包） |
| 文档版本 | docker 手册 2026-10-26 更新；安全强化手册 2026-08-04 更新 |

## 13. 踩坑清单（易错点）
1. **sudo 一致性**：docker login 与 docker pull 是否加 sudo 必须一致，否则报 no basic auth credentials。
2. **镜像名一致性**：compose.yaml 的 image 值必须与实际镜像一致；改 tag 只在名称不一致时做。
3. **防火墙别装错**：Debian 用 ufw，Redhat 系列用 firewalld，互不混装。
4. **容器 IP ≠ 宿主机 IP**：给 Pierre 的是容器内 IP（hostname -I，通常 172.20.0.5）。
5. **error mounting 报错**：映射文件缺失导致同名文件夹，需删文件夹、传文件、重启容器。
6. **临时排查文件**：docker cp 进容器的文件重启即失，程序文件必须打包进镜像。
7. **display_errors / debug 排查后必须关闭**，防止泄露报错细节。
8. **磁盘分区**：不建议单独 /home 分区且最大；/opt 或根分区要有足够空间放 eamic_storage。
9. **新版标准镜像**（2026-04-01 后）：v2、v2.js/v2_lf 已内置，compose.yaml 里注释掉映射，不要再上传。
10. **数据库账号**：连接账号是 mic00001@（% 或应用服务器 IP），不是 eamic 账号；密码须与 mic_admin.mic_site.site_password 一致。
