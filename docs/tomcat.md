# Tomcat 安全加固规范

## 1. 补丁安装

判断依据：1，查看 Tomcat 版本信息；2，关注官网 [Security Updates](https://tomcat.apache.org/security.html) 检查当前使用版本是否存在严重安全漏洞。

修复建议：更新 Tomcat 到对应版本的最新安全版本

## 2. 删除文档与示例程序

判断依据：在 Tomcat 部署程序的 webapps 目录下是否存在 docs 及 examples 目录，如果是则低于安全要求

修复建议：删除 webapps 目录下的 docs 及 examples 目录

## 3. 移除管理界面

判断依据：在 Tomcat 部署程序的 webapps 目录下是否存在 manager 及 host-manager 目录，如果是则低于安全要求

修复建议：在生成环境中，建议移除 manager 及 host-manager 目录

## 4. 设置 SHUTDOWN 字符

> Tomcat 监听 8005 端口用于接受关闭服务请求，用户可访问该端口并发送定义字符串关闭 Tomcat 服务
> 缺省值：默认监听 TCP 端口 8005 绑定于本地环回地址，默认关闭命令为 `SHUTDOWN`

判断依据：检查配置文件 conf/server.xml 中 8005 端口的 shutdown 字符串是否为简单字符串，如果是则低于安全要求

```
<Server port="8005" shutdown="SHUTDOWN">
```

修复建议：为 shutdown 属性设置复杂字符串，防止恶意用户猜测

## 5. 配置 Tomcat 运行账号

> 禁止 Tomcat 直接以具有管理员权限的账号启动，应新增普通账号用于启动 Tomcat 服务

判断依据：检查 Tomcat 启动脚本或服务，确认启动账号是否为普通账号，如果启动账号为管理员账号则低于安全要求

修复建议：1，新建普通权限账号；2，设置启动脚本或服务启动账号为新增账号；3，修改 Tomcat web 目录权限为新增账号

## 6. 禁止列目录

> 防止直接访问目录时由于找不到默认主页而列出目录下所有文件（默认值为 false）

判断依据：检查配置文件 conf/web.xml 中 listings 是否设置为 false

```
        <init-param>
            <param-name>listings</param-name>
            <param-value>false</param-value>
        </init-param>
```

修复建议：设置 listings 属性值为 false

## 7. 是否启用日志审计

判断依据：检查配置文件 conf/server.xml 以下配置是否在注释范围内（`<!-- *** -->`），如果配置被注释则低于安全要求

```
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
```

修复建议：确保访问日志已开启，并检查 logs 目录下是否有访问日志生成。