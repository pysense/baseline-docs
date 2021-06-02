# Linux 系统安全加固规范

## 1. 账号安全

提高账号安全的指导原则：

1. 增强密码复杂度（包含大小写字母、数字及特殊符号且至少为 8 位）
2. 定期更新密码

### 1.1. 检查系统账号是否存在弱密码

判断是否为弱密码可参考以下几个标准：

1. 密码位数少于8位
2. 密码为纯字母或数字组成，如：qwerty/123456/20080808
3. 密码为常见单词与数字的简单组合，如：root123/admin888

### 1.2. 检查是否存在空口令账号

判断依据：执行以下命令，如果输出不为空，则存在空口令账号

```bash
awk -F: '$2==""' /etc/shadow
```

修复建议：为空口令账号设置密码或锁定该账号（`passwd -l <username>`）

### 1.3. 配置密码策略

通过修改配置文件 /etc/login.defs 设置密码复杂度（长度）及密码最大有效期（只对修改后创建的用户生效）。

```
PASS_MAX_DAYS 60 #设置密码过期时间为 60 天，过期后需要重新设置密码
PASS_MIN_LEN  8  #设置密码最小长度为 8 位
```

可通过 `chage -l <username>` 查看账号信息。

### 1.4. 检查是否存在 UID 为 0 的非 root 账号

> 在 Linux 系统上 UID 0 代表特权账号，操作系统对于账号名是否为 root 并不关心

判断依据：执行以下命令，如果输出不为空，则存在 UID 为 0 的非 root 账号

```bash
awk -F: '$1!="root"&&$3==0' /etc/passwd
```

修复建议：如果不是自行创建的 UID 0 账号或者不是 root 重命名的账号，则应禁用该账号。

### 1.5. 检查用户环境变量安全性

1）检查 PATH 是否包含当前目录或父目录

判断依据：执行以下命令，如果输出不为空，则低于安全要求

```bash
echo $PATH | egrep '(^|:)(\.|:|$)'
```

2）检查 PATH 是否包含权限有问题的目录

判断依据：执行以下命令，如果输出不为空，则低于安全要求

```bash
find $(echo ${PATH//:/ }) -type d -perm -777 2> /dev/null
```

### 1.6. 检查是否存在可登录的可疑账号

判断依据：执行以下命令，查看是否存在可疑账号（非管理或用户账号）

```bash
sort -nk3 -t: /etc/passwd | egrep -v ":/(sbin/(nologin|shutdown|halt)|bin/(sync|false))$"
```

## 2. SSH 安全配置

### 2.1. 是否允许空口令登录

> PermitEmptyPasswords 缺省值为 no

判断依据：执行以下命令，如果输出不为空，则低于安全要求

```bash
grep -Ei "^\s*PermitEmptyPasswords\s+yes" /etc/ssh/sshd_config
```

修复建议：修改配置文件 /etc/ssh/sshd_config 增加以下配置：

```
PermitEmptyPasswords no
```

### 2.2. 是否允许 root 登录

> PermitRootLogin 缺省值为 yes，建议设置为 prohibit-password 禁用密码登录，但仍可通过密钥进行验证。

判断依据：执行以下命令，如果输出为空，则低于安全要求

```bash
grep -Ei "^\s*PermitRootLogin\s+[^y]+" /etc/ssh/sshd_config
```

修复建议：修改配置文件 /etc/ssh/sshd_config 增加以下配置：

```
PermitRootLogin prohibit-password
```

## 3. 文件、目录权限

### 3.1. 检查任何人都有写权限的目录

判断依据：执行以下命令，如果输出不为空，需要检查权限是否正确

```bash
find $(awk '$0~/^\s*[^#]/&&$2!="swap"{print$2}' /etc/fstab) -xdev -type d \( -perm -0002 -a ! -perm -1000 \)
```

### 3.2. 检查任何人都有写权限的文件

判断依据：执行以下命令，如果输出不为空，需要检查权限是否正确

```bash
find $(awk '$0~/^\s*[^#]/&&$2!="swap"{print$2}' /etc/fstab) -xdev -type f \( -perm -0002 -a ! -perm -1000 \)
```

### 3.3. 检查没有属主和属组的文件

判断依据：执行以下命令，如果输出不为空，需要检查权限是否正确

```bash
find $(awk '$0~/^\s*[^#]/&&$2!="swap"{print$2}' /etc/fstab) -xdev -nouser -o -nogroup
```

### 3.4. 检查可疑隐藏文件

判断依据：执行以下命令，如果输出不为空，需要检查输出文件是否异常

```bash
find $(awk '$0~/^\s*[^#]/&&$2!="swap"{print$2}' /etc/fstab) -xdev -name ".. *"
find $(awk '$0~/^\s*[^#]/&&$2!="swap"{print$2}' /etc/fstab) -xdev -name "...*"
```

### 3.5. 检查 SUID/SGID 文件

判断依据：执行以下命令，检查输出文件是否存在可疑文件

```bash
find $(awk '$0~/^\s*[^#]/&&$2!="swap"{print$2}' /etc/fstab) -xdev \( -perm -4000 -o -perm -2000 \)
```

## 4. 日志审计

### 4.1. 是否记录授权与安全相关的日志

判断依据：执行以下命令，如果输出为空，则低于安全要求

```bash
grep -Ei "^\s*authpriv\." /etc/rsyslog.conf
```

修复建议：修改配置文件 /etc/rsyslog.conf 增加以下配置：

```
authpriv.* /var/log/secure
```

### 4.2. 是否开启日志审计服务

1）检查 audit 内核模块是否启用（内核模块主要用来获取审计信息）

判断依据：执行以下命令，如果输出为 1，则审计为启用状态

```bash
auditctl -s | grep -i enabled
```

修复建议：通过以下命令改变 audit 内核模块状态

```bash
auditctl -e 1
```

2）检查 auditd 服务状态（用户态的守护进程主要用于收集信息和记录日志）

判断依据：执行以下命令，如果输出为空，则低于安全要求

```bash
systemctl status auditd | grep "active (running)"
```

修复建议：启用服务并启动服务

```bash
systemctl enable auditd
systemctl start auditd
```

### 4.3. 是否记录 cron 守护进程日志

判断依据：执行以下命令，如果输出为空，则低于安全要求

```bash
grep -Ei "^\s*cron\." /etc/rsyslog.conf
```

修复建议：修改配置文件 /etc/rsyslog.conf 增加以下配置：

```
cron.* /var/log/cron
```

### 4.4. 是否记录执行命令

判断依据：检查以下三个变量是否设置合适

```bash
echo $HISTFILE #记录执行命令的文件，检查是否为正常文件
echo $HISTFILESIZE #记录执行命令的文件的记录数，建议设置 1000 以上
echo $HISTSIZE #写入到 $HISTFILE 前的执行命令记录数，建议设置 1000 以上
```

### 4.5. 是否关闭 Core Dump

> Core Dump 文件可能包含敏感信息，建议关闭。

判断依据：执行以下命令，如果输出不为 0，则低于安全要求

```bash
ulimit -c
```

修复建议：通过设置 /etc/security/limits.conf 增加以下配置：

```
* soft core 0
* hard core 0
```

重新登录后生效或者执行 `ulimit -c 0` 在当前终端环境中生效（运行中的进程不受影响）。

## 5. 安全策略

### 5.1. 系统防火墙策略

判断依据：执行以下命令，判断系统防火墙是否启用，防火墙规则是否配置正确

```bash
systemctl status firewalld
iptables -nvL
```

### 5.2. CVM 安全组策略

> - 端口策略只开放需要开放的业务端口
> - 远程管理端口配置 IP 白名单

在[安全组 - 安全 - 私有网络 - 控制台](https://console.cloud.tencent.com/vpc/securitygroup?rid=1&rid=1)定义安全策略，并应用到实例中。

通过[实例端口验通 - 诊断工具 - 私有网络 - 控制台](https://console.cloud.tencent.com/vpc/helper?rid=1)检查端口开放状态。