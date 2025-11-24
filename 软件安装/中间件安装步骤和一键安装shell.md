

# MySQL

## 主要目录位置

| **安装目录**         | `/home/software/mysql-5.7.44-linux-glibc2.12-x86_64`         | MySQL 程序文件的根目录                   |
| -------------------- | ------------------------------------------------------------ | ---------------------------------------- |
| **数据目录**         | `/data/mysql`                                                | 存储数据库表、索引等数据文件             |
| **日志文件**         | `/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/mysqld.log` | 记录 MySQL 运行日志                      |
| **配置文件**         | `/etc/my.cnf`                                                | MySQL 的核心配置文件                     |
| **可执行文件目录**   | `/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin`     | 包含`mysql`、`mysqld`等命令              |
| **systemd 服务文件** | `/etc/systemd/system/mysqld.service`                         | 用于系统服务管理（启动、停止、开机自启） |
| **环境变量**         | 已将`/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin`添加到`/etc/profile` | 可直接在终端执行`mysql`等命令            |

## 安装步骤

- 启动服务：`systemctl start mysqld`
- 停止服务：`systemctl stop mysqld`
- 重启服务：`systemctl restart mysqld`
- 查看状态：`systemctl status mysqld`
- 远程连接：`mysql -uroot -p123456 -h 服务器IP -P 3306`

#### 步骤 1：安装依赖 + 创建目录

```
# 安装libaio依赖
rpm -ivh libaio-0.3.109-13.el7.x86_64.rpm --force --nodeps

# 创建所需目录
mkdir -p /home/software /data/mysql /var/run/mysqld /home/mysql
# 配置目录权限
chmod 777 /var/run/mysqld /home/mysql
```

#### 步骤 2：创建 mysql 用户组

```
# 新建mysql组（已存在则忽略报错）
groupadd -r mysql 2>/dev/null
# 新建mysql用户（已存在则忽略报错）
useradd -r -g mysql -s /sbin/nologin -d /home/mysql mysql 2>/dev/null
# 授权家目录
chown -R mysql:mysql /home/mysql
```

#### 步骤 3：解压安装包

```
# 解压到/home/software
tar -zxvf mysql-5.7.44-linux-glibc2.12-x86_64.tar.gz -C /home/software

# 授权安装目录
chown -R mysql:mysql /home/software/mysql-5.7.44-linux-glibc2.12-x86_64
chown -R mysql:mysql /data/mysql /var/run/mysqld

# 给执行文件加权限
chmod 755 /home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin/*
```

#### 步骤 4：生成配置文件

```
# 写入配置到/etc/my.cnf（覆盖原有文件）
cat > /etc/my.cnf << EOF
[mysqld]
bind-address=0.0.0.0
port=3306
basedir=/home/software/mysql-5.7.44-linux-glibc2.12-x86_64
datadir=/data/mysql
log-error=/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
socket=/tmp/mysql.sock
symbolic-links=0
lower_case_table_names=1

[mysql]
default-character-set=utf8
socket=/tmp/mysql.sock
EOF
```

#### 步骤 5：初始化数据库

```
# 清空数据目录（确保干净）
rm -rf /data/mysql/*
chown mysql:mysql /data/mysql

# 用mysql用户初始化（生成临时密码）
su - mysql -s /bin/bash -c "/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin/mysqld --defaults-file=/etc/my.cnf --initialize"

# 查看临时密码（复制保存，下一步用）
grep "temporary password" /home/software/mysql-5.7.44-linux-glibc2.12-x86_64/mysqld.log
```

#### 步骤6:启动mysql服务

```
# 用mysql用户启动服务
su - mysql -s /bin/bash -c "/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin/mysqld_safe --defaults-file=/etc/my.cnf &"

# 等待25秒（让服务启动稳定）
sleep 25

# 验证启动（有输出则成功）
ps -ef | grep -v grep | grep mysqld
netstat -tulpn | grep 3306
```

#### 步骤7:配置systemctl

```
# 写入服务文件
cat > /etc/systemd/system/mysqld.service << EOF
[Unit]
Description=MySQL 5.7.44 Server
After=network.target

[Service]
Type=forking
User=mysql
Group=mysql
ExecStart=/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin/mysqld_safe --defaults-file=/etc/my.cnf
ExecStop=/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin/mysqladmin -uroot -p123456 --socket=/tmp/mysql.sock shutdown
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

# 重载配置+设置开机自启
systemctl daemon-reload
systemctl enable mysqld
```

#### 步骤 8：修改密码 + 开启远程访问

```
#首先，使用临时密码登录 MySQL 服务器
/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin/mysql -uroot -p --connect-expired-password


#修改 root 用户的密码
ALTER USER 'root'@'localhost' IDENTIFIED BY '123456';

#授权 root 用户允许远程连接
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123456' WITH GRANT OPTION;
#刷新权限，使更改生效
FLUSH PRIVILEGES;
```

#### 步骤 9：配置环境变量（直接用 mysql 命令）

```
# 添加环境变量
echo "export PATH=\$PATH:/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin" >> /etc/profile
echo "export PATH=\$PATH:/home/software/mysql-5.7.44-linux-glibc2.12-x86_64/bin" >> ~/.bashrc

# 生效环境变量
source /etc/profile
source ~/.bashrc
```



## 一键安装shell脚本

```
#!/bin/bash
PASSWORD="123456"
TAR_FILE="mysql-5.7.44-linux-glibc2.12-x86_64.tar.gz"
INSTALL_BASE="/home/software"
DATA_DIR="/data/mysql"
SERVICE_FILE="/etc/systemd/system/mysqld.service"
MYSQL_CNF_TMP="/tmp/mysql_tmp.cnf"

# 输出函数
info() { echo -e "\033[32m[INFO] $1\033[0m"; }
warn() { echo -e "\033[33m[WARN] $1\033[0m"; }
error() { echo -e "\033[31m[ERROR] $1\033[0m"; exit 1; }

# ======================================
# 1. 基础检查
# ======================================
info "===== 1. 基础环境检查 ====="
[ $EUID -ne 0 ] && error "请以root用户执行"
[ ! -f "$TAR_FILE" ] && error "未找到安装包 $TAR_FILE"
info "✅ 基础检查通过"

# ======================================
# 2. 强力清理残留
# ======================================
info ""
info "===== 2. 强力清理残留 ====="
# 终止所有MySQL进程
pkill -f mysqld 2>/dev/null || true
pkill -f mysqld_safe 2>/dev/null || true
for ((i=0; i<5; i++)); do
    MYSQL_PIDS=$(ps -ef | grep -v grep | grep -E "mysqld|mysqld_safe" | awk '{print $2}')
    [ -z "$MYSQL_PIDS" ] && break
    kill -9 $MYSQL_PIDS 2>/dev/null || true
    sleep 2
done
[ -n "$(ps -ef | grep -v grep | grep -E "mysqld|mysqld_safe")" ] && error "残留进程无法终止"

# 清理端口和文件
PORT_PID=$(netstat -tulpn | grep :3306 | awk '{print $7}' | cut -d '/' -f1)
[ -n "$PORT_PID" ] && kill -9 $PORT_PID 2>/dev/null && sleep 2
UNTAR_DIR=$(tar -ztf "$TAR_FILE" 2>/dev/null | head -1 | cut -d '/' -f1)
MYSQL_HOME="${INSTALL_BASE}/${UNTAR_DIR}"
rm -rf "$MYSQL_HOME" "$DATA_DIR"/* "$MYSQL_CNF_TMP" /tmp/mysql.sock* 2>/dev/null
info "✅ 残留清理完成"

# ======================================
# 3. 安装依赖+创建环境
# ======================================
info ""
info "===== 3. 安装依赖+创建环境 ====="
for rpm in ./*.rpm; do
    [ -f "$rpm" ] && { info "安装依赖：$rpm"; rpm -ivh "$rpm" --force --nodeps 2>/dev/null; }
done
mkdir -p "$INSTALL_BASE" "$DATA_DIR" "/var/run/mysqld" "/home/mysql"
chmod 777 "/var/run/mysqld" "/home/mysql"
grep -q "mysql" /etc/group || groupadd -r mysql
id -u mysql > /dev/null 2>&1 || useradd -r -g mysql -s /sbin/nologin -d /home/mysql mysql
chown -R mysql:mysql /home/mysql
info "✅ 依赖+环境就绪"

# ======================================
# 4. 解压+权限配置
# ======================================
info ""
info "===== 4. 解压+权限配置 ====="
tar -zxvf "$TAR_FILE" -C "$INSTALL_BASE" > /dev/null || error "解压失败"
MYSQL_HOME="${INSTALL_BASE}/${UNTAR_DIR}"
MYSQL_BIN="${MYSQL_HOME}/bin"
LOG_FILE="${MYSQL_HOME}/mysqld.log"
chown -R mysql:mysql "$MYSQL_HOME" "$DATA_DIR" "/var/run/mysqld"
touch "$LOG_FILE" && chown mysql:mysql "$LOG_FILE"
chmod 755 "$MYSQL_BIN"/*
info "✅ 解压完成（目录：$MYSQL_HOME）"

# ======================================
# 5. 生成配置文件
# ======================================
info ""
info "===== 5. 生成配置文件 ====="
cat > /etc/my.cnf << EOF
[mysqld]
bind-address=0.0.0.0
port=3306
basedir=$MYSQL_HOME
datadir=$DATA_DIR
log-error=$LOG_FILE
pid-file=/var/run/mysqld/mysqld.pid
socket=/tmp/mysql.sock
symbolic-links=0
lower_case_table_names=1

[mysql]
default-character-set=utf8
socket=/tmp/mysql.sock
EOF
info "✅ 配置文件生成"

# ======================================
# 6. 数据库初始化
# ======================================
info ""
info "===== 6. 数据库初始化 ====="
rm -rf "$DATA_DIR"/* && chown mysql:mysql "$DATA_DIR"
su - mysql -s /bin/bash -c "$MYSQL_BIN/mysqld --defaults-file=/etc/my.cnf --initialize" || {
    error "初始化失败，查看日志：$LOG_FILE"
}
TEMP_PWD=$(grep "temporary password" "$LOG_FILE" | awk '{print $NF}')
[ -z "$TEMP_PWD" ] && error "未获取到临时密码，查看日志：$LOG_FILE"
info "✅ 初始化完成，临时密码：$TEMP_PWD"

# ======================================
# 7. 启动服务
# ======================================
info ""
info "===== 7. 启动MySQL服务 ====="
[ -n "$(ps -ef | grep -v grep | grep mysqld)" ] && error "进程冲突"
su - mysql -s /bin/bash -c "$MYSQL_BIN/mysqld_safe --defaults-file=/etc/my.cnf &"
sleep 25

if ps -ef | grep -v grep | grep mysqld &> /dev/null && \
   netstat -tulpn | grep 3306 &> /dev/null && \
   [ -S "/tmp/mysql.sock" ]; then
    info "✅ MySQL服务启动成功"
else
    error "服务启动失败，查看日志：$LOG_FILE"
fi
cat > "$SERVICE_FILE" << EOF
[Unit]
Description=MySQL 5.7.44 Server
After=network.target

[Service]
Type=forking
User=mysql
Group=mysql
ExecStart=$MYSQL_BIN/mysqld_safe --defaults-file=/etc/my.cnf
ExecStop=$MYSQL_BIN/mysqladmin -uroot -p$PASSWORD --socket=/tmp/mysql.sock shutdown
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
systemctl daemon-reload
systemctl enable mysqld &> /dev/null
info "✅ 开机自启配置完成"

# ======================================
# 8. 配置密码+远程访问（核心修复：兼容MySQL 5.7 SQL语法）
# ======================================
info ""
info "===== 8. 配置密码+远程访问 ====="
# 临时配置文件传递密码
cat > "$MYSQL_CNF_TMP" << EOF
[client]
user=root
password=$TEMP_PWD
socket=/tmp/mysql.sock
EOF
chmod 600 "$MYSQL_CNF_TMP"

# 修复语法：MySQL 5.7 不支持 UNINSTALL PLUGIN IF EXISTS，先查询再卸载（忽略不存在错误）
"$MYSQL_BIN/mysql" --defaults-extra-file="$MYSQL_CNF_TMP" --connect-expired-password -e "
-- 先查询插件是否存在，存在则卸载（兼容5.7语法）
SELECT PLUGIN_NAME FROM INFORMATION_SCHEMA.PLUGINS WHERE PLUGIN_NAME = 'validate_password';
" 2>/dev/null | grep -q "validate_password" && {
    "$MYSQL_BIN/mysql" --defaults-extra-file="$MYSQL_CNF_TMP" --connect-expired-password -e "
    UNINSTALL PLUGIN validate_password;
    " 2>/dev/null
}

# 执行密码修改和授权（无语法错误）
"$MYSQL_BIN/mysql" --defaults-extra-file="$MYSQL_CNF_TMP" --connect-expired-password -e "
ALTER USER 'root'@'localhost' IDENTIFIED BY '$PASSWORD';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '$PASSWORD' WITH GRANT OPTION;
FLUSH PRIVILEGES;
" 2>/tmp/mysql_error.log || {
    rm -f "$MYSQL_CNF_TMP"
    error "密码配置失败，错误日志：/tmp/mysql_error.log，手动执行：
$MYSQL_BIN/mysql -uroot -p\"$TEMP_PWD\" --connect-expired-password --socket=/tmp/mysql.sock
然后执行：
ALTER USER 'root'@'localhost' IDENTIFIED BY '$PASSWORD';
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '$PASSWORD';
FLUSH PRIVILEGES;
"
}
rm -f "$MYSQL_CNF_TMP"
info "✅ 密码修改完成（密码：$PASSWORD）"
info "✅ 远程访问已授权"

# ======================================
# 9. 环境变量+最终验证
# ======================================
info ""
info "===== 9. 环境变量+最终验证 ====="
echo "export PATH=\$PATH:$MYSQL_BIN" >> /etc/profile
echo "export PATH=\$PATH:$MYSQL_BIN" >> ~/.bashrc
source /etc/profile && source ~/.bashrc

mysql -uroot -p"$PASSWORD" --socket=/tmp/mysql.sock -e "SELECT VERSION();" &> /dev/null && {
    info "✅ 本地连接测试成功！"
} || {
    error "本地连接失败，手动执行：mysql -uroot -p$PASSWORD --socket=/tmp/mysql.sock"
}

# 最终提示
info ""
info "======================================"
info "🎉 MySQL 5.7.44 安装全部完成！"
info "======================================"
info "访问信息："
info "  IP：$(hostname -I | awk '{print $1}')"
info "  端口：3306"
info "  账号：root"
info "  密码：$PASSWORD"
info "  连接命令：mysql -uroot -p$PASSWORD"
info "服务管理："
info "  启动：systemctl start mysqld"
info "  停止：systemctl stop mysqld"
info "  重启：systemctl restart mysqld"
info "======================================"
```





# Redis

### 配置文件

```

vim  /usr/local/redis/redis.conf 
bind	127.0.0.1	0.0.0.0	允许所有 IP 连接（生产环境建议绑定特定 IP）
protected-mode	yes	no	关闭保护模式，允许远程连接
daemonize	no	yes	开启后台运行，否则关闭终端会停止服务
port	6379	(可选修改)	建议修改默认端口提高安全性
```

### 安装步骤

```
[root@localhost ~]# cd /usr/local
[root@localhost ~]# tar -zxvf redis-6.2.14.tar.gz
[root@localhost local]# mv redis-6.2.14 redis
#进入到redis目录中执行：
[root@localhost local]# cd /usr/local/redis
#编译redis
[root@localhost redis]# make install 
[root@localhost redis]# cd /usr/local/bin

./redis-server /usr/local/redis/redis.conf 
```

### 一键安装shell

- 二进制文件：`/home/software/redis/bin`
- 配置文件：`/home/software/redis/redis.conf`
- 数据目录：`/home/software/redis/data`
- 日志文件：`/home/software/redis/logs/redis.log`

```
#!/bin/bash
# Redis 6.2.14 一键安装脚本（安装路径：/home/software/redis）
# 依赖：gcc（若未安装需手动执行 yum install -y gcc 或 apt install gcc）

# 定义核心变量（安装到 /home/software 下）
REDIS_TAR="redis-6.2.14.tar.gz"
INSTALL_BASE="/home/software"
REDIS_DIR="${INSTALL_BASE}/redis"  # Redis 主目录
CONF_FILE="${REDIS_DIR}/redis.conf"  # 配置文件路径
BIN_DIR="${REDIS_DIR}/bin"  # 二进制文件目录（不污染系统路径）
SERVICE_FILE="/etc/systemd/system/redis.service"  # systemd 服务文件

# 颜色输出函数
info() { echo -e "\033[32m[INFO] $1\033[0m"; }
error() { echo -e "\033[31m[ERROR] $1\033[0m"; exit 1; }

# 1. 检查依赖（gcc）
info "===== 1. 检查编译依赖 ====="
if ! command -v gcc &> /dev/null; then
    error "未检测到 gcc 编译器，请先执行：
CentOS：yum install -y gcc
Ubuntu：apt install gcc
安装后再运行脚本"
fi

# 2. 检查安装包
info "===== 2. 检查 Redis 安装包 ====="
if [ ! -f "$REDIS_TAR" ]; then
    error "未找到 Redis 安装包 ${REDIS_TAR}，请将其放在当前目录"
fi

# 3. 清理残留（若之前安装过）
info "===== 3. 清理残留文件 ====="
rm -rf "$REDIS_DIR"  # 删除旧的 Redis 目录
rm -f "$SERVICE_FILE"  # 删除旧的 systemd 服务文件
info "残留清理完成"

# 4. 解压安装包到 /home/software
info "===== 4. 解压安装包 ====="
mkdir -p "$INSTALL_BASE"  # 确保 /home/software 目录存在
tar -zxvf "$REDIS_TAR" -C "$INSTALL_BASE" > /dev/null || error "解压 ${REDIS_TAR} 失败"
mv "${INSTALL_BASE}/redis-6.2.14" "$REDIS_DIR" || error "重命名 Redis 目录失败"
info "解压完成，Redis 主目录：${REDIS_DIR}"

# 5. 编译安装（指定安装路径为 REDIS_DIR）
info "===== 5. 编译安装 Redis ====="
cd "$REDIS_DIR" || error "进入 Redis 目录 ${REDIS_DIR} 失败"
make PREFIX="$REDIS_DIR" install > /dev/null || error "编译安装失败，请检查 gcc 版本或安装包完整性"
info "编译安装完成，二进制文件路径：${BIN_DIR}"

# 6. 修改 Redis 配置文件
info "===== 6. 配置 Redis ====="
# 允许所有 IP 连接（生产环境建议改为具体 IP）
sed -i 's/bind 127.0.0.1/bind 0.0.0.0/' "$CONF_FILE"
# 关闭保护模式（允许远程连接）
sed -i 's/protected-mode yes/protected-mode no/' "$CONF_FILE"
# 开启后台运行
sed -i 's/daemonize no/daemonize yes/' "$CONF_FILE"
# 指定数据存储目录（在 Redis 主目录下创建 data 目录）
mkdir -p "${REDIS_DIR}/data"
sed -i "s#dir ./#dir ${REDIS_DIR}/data/#" "$CONF_FILE"
# 指定日志文件路径（在 Redis 主目录下创建 logs 目录）
mkdir -p "${REDIS_DIR}/logs"
sed -i "s#logfile \"\"#logfile \"${REDIS_DIR}/logs/redis.log\"#" "$CONF_FILE"
info "配置完成，关键配置："
info "  监听 IP：0.0.0.0（允许所有远程连接）"
info "  数据目录：${REDIS_DIR}/data"
info "  日志文件：${REDIS_DIR}/logs/redis.log"

# 7. 配置 systemd 服务（使用绝对路径，避免环境变量问题）
info "===== 7. 配置 systemd 服务 ====="
cat > "$SERVICE_FILE" << EOF
[Unit]
Description=Redis 6.2.14 Server（安装路径：${REDIS_DIR}）
After=network.target

[Service]
Type=forking
User=root
Group=root
# 启动命令（使用绝对路径）
ExecStart=${BIN_DIR}/redis-server ${CONF_FILE}
# 停止命令（使用绝对路径，通过 redis-cli 关闭）
ExecStop=${BIN_DIR}/redis-cli shutdown
# 重启策略
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# 重载 systemd 配置
systemctl daemon-reload
info "systemd 服务配置完成：${SERVICE_FILE}"

# 8. 启动 Redis 并设置开机自启
info "===== 8. 启动 Redis 服务 ====="
systemctl enable redis &> /dev/null || error "设置 Redis 开机自启失败"
systemctl start redis || error "启动 Redis 服务失败，请执行 systemctl status redis 查看详情"
sleep 3  # 等待服务稳定启动

# 9. 验证安装结果
info "===== 9. 验证安装 ====="
# 检查服务状态
if systemctl is-active --quiet redis; then
    info "✅ Redis 服务启动成功（systemd 状态：active）"
else
    error "❌ Redis 服务启动失败，状态：$(systemctl is-active redis)"
fi

# 检查端口监听（默认 6379）
if netstat -tulpn | grep -q "redis-server"; then
    info "✅ Redis 端口 6379 监听正常"
else
    warn "⚠️  未检测到 Redis 端口监听，请检查配置文件"
fi

# 检查版本（使用绝对路径执行 redis-cli）
REDIS_VERSION=$("${BIN_DIR}/redis-cli" INFO server | grep "redis_version" | awk -F: '{print $2}' | tr -d '[:space:]')
if [ "$REDIS_VERSION" = "6.2.14" ]; then
    info "✅ Redis 版本验证通过：${REDIS_VERSION}"
else
    warn "⚠️  Redis 版本异常，实际版本：${REDIS_VERSION}"
fi

# 10. 输出最终信息
info -e "\n======================================"
info "🎉 Redis 6.2.14 安装完成！"
info "======================================"
info "📌 安装路径：${REDIS_DIR}"
info "📌 配置文件：${CONF_FILE}"
info "📌 数据目录：${REDIS_DIR}/data"
info "📌 日志文件：${REDIS_DIR}/logs/redis.log"
info "📌 二进制文件：${BIN_DIR}"
info -e "\n🔧 服务管理命令："
info "  启动：systemctl start redis"
info "  停止：systemctl stop redis"
info "  重启：systemctl restart redis"
info "  状态：systemctl status redis"
info "  开机自启：已启用（systemctl enable redis）"
info -e "\n📡 连接命令："
info "  本地连接：${BIN_DIR}/redis-cli"
info "  远程连接：${BIN_DIR}/redis-cli -h 服务器IP -p 6379"
info -e "\n⚠️  注意："
info "  1. 生产环境建议设置密码（修改 ${CONF_FILE} 中的 requirepass 配置）"
info "  2. 限制连接 IP（将 bind 0.0.0.0 改为具体的客户端 IP）"
info "  3. 防火墙开放 6379 端口（CentOS：firewall-cmd --add-port=6379/tcp --permanent && firewall-cmd --reload）"
info "======================================"
```







```



```

# tql

```
unzip jdk.zip
tar -zxvf TLQ8.tar.gz
tar -zxvf tlq.sh

for file in *.gz; do gzip -d "$file"; done
```



```
[RcvProcess]		# 接收进程小节
[RcvProcessRecord]		# 
ListenPort = 10071		# 监听端口，端口号需大于1024



[RemoteQue]		# 远程队列单元小节
[RemoteQueRecord]	
DestQueName = lq		# 目的队列名
[RemoteQueRecord]
DestQueName = lq_1		# 目的队列名









[SendProcess]		# 发送进程小节
[SendProcessRecord]		# 
[SendConnRecord]		# 发送连接小节
HostName = 10.32.9.165		# 被连接节点的IP或主机名或节点名
ConnPort = 10071		# 被连接节点的端口号，端口号需大于1024




[LocalQue]		# 本地队列单元小节
[LocalQueRecord]		# 
LocalQueName = lq		# 本地队列名
[LocalQueRecord]		# 
LocalQueName = lq_1		# 本地队列名
```

```
#!/bin/sh
#
# This script will be executed *after* all the other init scripts.
# You can put your own initialization stuff in here if you don't
# want to do the full Sys V style init stuff.
TLQHOMEDIR=/home/TLQ8; export TLQHOMEDIR
TLQLICENSEDIR=$TLQHOMEDIR; export TLQLICENSEDIR
TLQCONFDIR=$TLQHOMEDIR/etc; export TLQCONFDIR
TLQLOGDIR=$TLQHOMEDIR/log; export TLQLOGDIR
TLQSNDFILESDIR=$TLQHOMEDIR/sndfiles; export TLQSNDFILESDIR
TLQRCVFILESDIR=$TLQHOMEDIR/rcvfiles; export TLQRCVFILESDIR
TLQMSGDIR=$TLQHOMEDIR/msg; export TLQMSGDIR


PATH=$TLQHOMEDIR/bin:$TLQHOMEDIR/samples/bin:.:$PATH
export PATH


if [ `env|grep -c CLASSPATH` -eq 0 ]; then
CLASSPATH=$TLQHOMEDIR/java/lib/tlclient.jar:$TLQHOMEDIR/java/lib/TLQRemoteApi.jar:$TLQHOMEDIR/java/conf:$TLQHOMEDIR/java/lib/javaee.jar:$TLQHOMEDIR/java/lib/TongJMS.jar:.
else
CLASSPATH=$TLQHOMEDIR/java/lib/tlclient.jar:$TLQHOMEDIR/java/lib/TLQRemoteApi.jar:$TLQHOMEDIR/java/conf:$TLQHOMEDIR/java/lib/javaee.jar:$TLQHOMEDIR/java/lib/TongJMS.jar:.:$CLASSPATH
fi
export CLASSPATH


if [ `env|grep -c LD_LIBRARY_PATH` -eq 0 ]; then
LD_LIBRARY_PATH=$TLQHOMEDIR/lib  #for DEC SCO SUN  LINUX
else
LD_LIBRARY_PATH=$TLQHOMEDIR/lib:$LD_LIBRARY_PATH  #for DEC SCO SUN  LINUX
fi
export LD_LIBRARY_PATH


if [ `env|grep -c LIBPATH` -eq 0 ]; then
LIBPATH=$TLQHOMEDIR/lib         #for IBM
else
LIBPATH=$TLQHOMEDIR/lib:$LIBPATH          #for IBM
fi
export LIBPATH


if [ `env|grep -c SHLIB_PATH` -eq 0 ]; then
SHLIB_PATH=$TLQHOMEDIR/lib         #for IBM
else
SHLIB_PATH=$TLQHOMEDIR/lib:$SHLIB_PATH          #for HP
fi
export SHLIB_PATH

#tlqbridge set tlq6 client envionment
#TCLIHOMEDIR=/home/tong/TLQCli63; export TCLIHOMEDIR
#TCLICONFDIR=$TCLIHOMEDIR/etc; export TCLICONFDIR
#TCLIFILESDIR=$TCLIHOMEDIR/files; export TCLIFILESDIR
#TCLILOGDIR=$TCLIHOMEDIR/log; export TCLILOGDIR
#
#LD_LIBRARY_PATH=$TCLIHOMEDIR/lib:$LD_LIBRARY_PATH  #for DEC SCO SUN  LINUX
#export LD_LIBRARY_PATH
#LIBPATH=$TCLIHOMEDIR/lib:$LIBPATH         #for IBM
#export LIBPATH
#SHLIB_PATH=$TCLIHOMEDIR/lib:$SHLIB_PATH       #for HP
#export SHLIB_PATH

tlq -cstart
tlqremote -cstart

```

# minio

```
# 创建minio目录 并给许minio文件执行权限
[root@localhost ~]# mkdir -p /usr/local/minio/bin
[root@localhost ~]#chomod 777/usr/local/minio/bin/minio
# 创建桶目录
[root@localhost ~]# mkdir -p /usr/local/minio/data
```





```
# nohup：忽略终端挂断信号，确保终端关闭后MinIO仍能后台运行
nohup \
  ./minio \                # 执行当前目录下的MinIO可执行文件（需提前赋予执行权限）
  server \                 # MinIO的核心命令，用于启动对象存储服务
  /data/minio \            # 指定MinIO的数据存储目录（所有上传的文件数据会保存在此目录）
  --address ":8030" \      # 自定义API端口（后台服务端口，用于S3协议交互，如客户端/SDK连接）为8030
  --console-address ":8040" \  # 自定义Web控制台端口（浏览器访问的图形化管理界面）为8040
  > /home/software/minio/minio.log \  # 将程序的标准输出（stdout，如正常运行日志）重定向到指定日志文件
  2>&1 \                    # 将标准错误（stderr，如错误信息）重定向到标准输出，与正常日志合并写入同一文件
  &                         # 将程序放入后台运行，不阻塞当前终端（可继续输入其他命令）
```

```
# 假设已设置环境变量（可选，若修改了凭据）
nohup ./minio server /data/minio --address ":8030" --console-address ":8040" > /home/software/minio/minio.log 2>&1 &
```

# mysql



```
tar -zxvf mysql-5.7.44-linux-glibc2.12-x86_64.tar.gz




.tar.gz → tar -zxvf
.tar.xz → tar -xJvf
.zip → unzip
```

mysql.conf

```
[mysqld]
# 基础默认配置（无需修改路径）
bind-address=0.0.0.0
port=3306
basedir=/home/software/mysql  # 软链接目录，符合默认规范
datadir=/data/mysql    # 默认数据目录
log-error=/home/software/mysql/mysqld.log  # 默认错误日志
pid-file=/var/run/mysqld/mysqld.pid  # 默认PID文件
socket=/tmp/mysql.sock    # 默认socket路径（客户端自动识别）

# 安全与性能配置
symbolic-links=0
explicit_defaults_for_timestamp=1  # 解决TIMESTAMP警告
max_connections=100
innodb_buffer_pool_size=128M

[mysql]
default-character-set=utf8  # 客户端默认字符集
# socket无需额外配置，默认即为/tmp/mysql.sock
```

### 初始化

```
# 进入默认安装目录的bin目录
cd /usr/local/mysql/bin/

# 执行初始化
./mysqld --defaults-file=/etc/my.cnf --initialize
```

### 查看密码

```
grep "temporary password" /home/software/mysql/mysqld.log

```

### 启动mysql

```
/usr/local/mysql/bin/mysqld_safe --defaults-file=/etc/my.cnf --user=root &
```

### 修改密码

```
/home/software/mysql/bin/mysql -uroot -p
ALTER USER 'root'@'localhost' IDENTIFIED BY '123456';


```

### 远程访问

```
# 授权所有IP访问（生产环境建议限制IP，如192.168.1.%）
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123456' WITH GRANT OPTION;

# 刷新权限
FLUSH PRIVILEGES;
```

# nginx

#### 1. 前置准备

- 创建目标目录：`mkdir -p /home/software/nginx`（所有组件最终安装于此）

- 压缩包存放目录：保持 `/home/software/nginx`（与目标目录一致，无需额外移动文件）

- 设置 Perl 环境变量（避免系统默认 Perl 干扰）：

  ```bash
  export PATH="/home/software/nginx/perl/bin:$PATH"
  export PERL5LIB="/home/software/nginx/perl/lib/perl5:$PERL5LIB"
  ```

  

#### 2. 依赖安装（统一安装到 `/home/software/nginx/[依赖名]`）

- Perl 及模块

  ```bash
  tar -zxvf perl-5.38.2.tar.gz && cd perl-5.38.2
  ./Configure -des -Dprefix=/home/software/nginx/perl  # 安装到nginx下的perl子目录
  make && make install && cd ..
  # 安装8个Perl模块（用上面的perl路径执行，模块自动安装到perl/lib下）
  ```

  

- pcre2/zlib/OpenSSL

  （保留源码目录，供 Nginx 配置识别）：

  ```bash
  # pcre2示例（zlib、OpenSSL步骤一致，仅替换包名和路径）
  tar -zxvf pcre2-10.44.tar.gz && cd pcre2-10.44
  ./configure --prefix=/home/software/nginx/pcre2  # 安装到nginx下的pcre2子目录
  make && make install && cd ..  # 保留pcre2-10.44源码目录
  ```

  

#### 3. Nginx 编译安装（直接安装到 `/home/software/nginx`）

```bash
tar -zxvf nginx-1.26.2.tar.gz && cd nginx-1.26.2
./configure \
--prefix=/home/software/nginx \  # 最终安装目录（直接到nginx，无需nginx_install子目录）
--with-pcre=../pcre2-10.44 \    # 指向pcre2源码目录（上级目录下）
--with-openssl=../openssl-3.2.2 \  # 指向OpenSSL源码目录
--with-zlib=../zlib-1.3.1 \      # 指向zlib源码目录
--with-http_ssl_module \
--with-http_gzip_static_module
make && make install && cd ..
```

#### 4. 启动与验证（路径简化）

```bash
# 启动Nginx（路径更简洁）
/home/software/nginx/sbin/nginx
# 验证进程
ps -ef | grep nginx
# 验证访问
curl http://localhost
```

- 启动：`/home/software/nginx/sbin/nginx`
- 停止：`/home/software/nginx/sbin/nginx -s stop`
- 重启：`/home/software/nginx/sbin/nginx -s reload`
- 配置文件路径：`/home/software/nginx/conf/nginx.conf`
- 日志路径：`/home/software/nginx/logs/`