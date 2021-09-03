# Windows 系统安全加固规范

## 1. 账号安全

### 1.1. 检查系统账号是否存在弱密码

判断是否为弱密码可参考以下几个标准：

1. 密码位数少于8位
2. 密码为纯字母或数字组成，如：qwerty/123456/20080808
3. 密码为常见单词与数字的简单组合，如：root123/admin888

### 1.2. 检查是否禁用 Guest 账号

> 实施目的：禁用来宾账号，提高系统安全性
> 实施风险：低

判断依据：执行以下命令，如果输出中“帐户启用”不为“No”，则低于安全要求

```
net user guest
```

修复建议：执行以下命令，禁用 Guest 账号

```
net user guest /active:no
```

### 1.3. 配置密码策略

> 实施目的：防止系统弱口令的存在，减少安全隐患
> 实施风险：低

判断依据：执行 `secpol` 打开“本地安全策略” > 本地策略 > 密码策略，查看以下项目设置是否符合预期，否则低于安全要求

```
密码必须符合复杂性要求 | 已启用
密码长度最小值 | 8
强制密码历史 | 5
```

修复建议：执行 `secpol` 打开“本地安全策略” > 本地策略 > 密码策略，更改相关项目设置。

## 2. RDP 安全配置


## 3. 共享文件夹及访问权限

### 3.1. 检查是否关闭默认共享

> 实施目的：非域环境中，关闭 Windows 默认共享，例如 C$、D$，防止攻击者非法对系统的硬盘进行访问
> 实施风险：低

判断依据：执行以下命令，如果输出不为 0x0，或提示注册表值不存在，则低于安全风险

```
reg query "HKLM\System\CurrentControlSet\Services\LanmanServer\Parameters" /v AutoShareServer
```

修复建议：执行以下命令关闭默认共享（需重启后生效）

```
reg add "HKLM\System\CurrentControlSet\Services\LanmanServer\Parameters" /v AutoShareServer /t REG_DWORD /d 0
```

### 3.2. 检查共享文件夹访问权限

> 实施目的：设置共享文件夹访问权限，只允许授权的账户访问共享文件夹
> 实施风险：低

判断依据：执行以下命令，判断是否存在共享文件夹，如果存在则判断共享权限是否正确

```
net share # 查看所有共享
net share <sharename> # 查看某个共享权限
```

修复建议：执行以下命令停止共享，或执行 `compmgmt.msc` 打开“计算机管理” > 系统工具 > 共享文件夹 > 共享，根据业务需要设置，不建议提供 Everyone 权限。

```
net share <sharename> /delete
```

## 4. 日志审计

### 4.1. 审核登录事件

> 实施目的：设置审核策略，记录系统重要的事件日志
> 实施风险：低

判断依据：执行 `secpol` 打开“本地安全策略” > 本地策略 > 审核策略，查看“安全设置”是否设置为审核“成功”与“失败”，如果显示“无审核”，则低于安全要求

修复建议：执行 `secpol` 打开“本地安全策略” > 本地策略 > 审核策略，更改审核策略为“成功”与“失败”都审核。

### 4.2. 审核进程创建

> 实施目的：记录进程创建详细信息
> 实施风险：低

1）启用审核进程创建

判断依据：执行 `gpedit.msc`，查看：计算机配置 > Windows 设置 > 安全设置 > 高级审核策略配置 > 系统审核策略 > 详细跟踪 > 审核进程创建，如果显示“未配置”，则低于安全要求

修复建议：执行 `gpedit.msc`，查看：计算机配置 > Windows 设置 > 安全设置 > 高级审核策略配置 > 系统审核策略 > 详细跟踪 > 审核进程创建，设置审核“成功”及“失败”事件

2）在进程创建事件中包含命令行

判断依据：执行 `gpedit.msc`，查看：计算机配置 > 管理模板 > 系统 > 审核过程创建 > 在过程创建事件中加入命令行，如果显示“未配置”，则低于安全要求

修复建议：执行 `gpedit.msc`，查看：计算机配置 > 管理模板 > 系统 > 审核过程创建 > 在过程创建事件中加入命令行，设置为“已启用”

### 4.3. 检查日志文件大小

> 实施目的：提高日志文件最大大小，防止旧日志过早被覆盖
> 实施风险：低

判断依据：执行以下命令查看某个日志文件配置的最大大小（单位为 Byte），如果配置大小小于 20M，则低于安全要求

```
wevtutil gl system | findstr maxSize
wevtutil gl security | findstr maxSize
```

修复建议：执行以下命令查更新日志文件最大大小为 20M（建议不小于 20M），或者通过执行 `compmgmt.msc` 打开“计算机管理” > 系统工具 > 事件查看器 > Windows 日志，手动设置日志最大大小。

```
wevtutil sl system /maxsize:20971520
wevtutil sl security /maxsize:20971520
```

## 5. 更新系统

> 实施目的：修复系统漏洞，降低被攻击、渗透的风险
> 实施风险：中

判断依据：打开“开始”菜单 > 设置 > 更新和安全，查看是否已安装所有更新。或者在 Powershell 中执行以下命令，查看是否存在更新：

```powershell
Install-Module PSWindowsUpdate #安装 Windows Update 模块
Get-WindowsUpdate #检查并下载更新
```

修复建议：系统盘创建快照备份，并安装 Windows 更新。

## 6. 安全策略

### 6.1. 系统防火墙策略

1）检查 Windows Defender 防火墙是否启用

判断依据：执行以下命令，如果输出结果为“关闭”，则低于安全要求

```
netsh advfirewall show publicprofile | findstr 状态
```

修复方法：在确保防火墙策略配置正确后，执行以下命令启用 Windows Defender 防火墙

```
netsh advfirewall set publicprofile state on
```

2）检查 Windows Defender 防火墙策略是否配置正确

判断依据：执行 `wf` 打开 Windows Defender 防火墙，检查防火墙规则是否配置正确。

### 6.2. CVM 安全组策略

> - 端口策略只开放需要开放的业务端口
> - 远程管理端口配置 IP 白名单

在[安全组 - 安全 - 私有网络 - 控制台](https://console.cloud.tencent.com/vpc/securitygroup?rid=1&rid=1)定义安全策略，并应用到实例中。

通过[实例端口验通 - 诊断工具 - 私有网络 - 控制台](https://console.cloud.tencent.com/vpc/helper?rid=1)检查端口开放状态。

## 7. 其他安全配置

### 7.1. 检查是否开启防病毒软件
