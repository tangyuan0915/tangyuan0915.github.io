Grafana 是数据可视化展示平台，可连接包括 Zabbix 在内的多种数据源。Zabbix 是一套企业级的开源监控解决方案，包含监控、报警和数据存储等功能，Zabbix 通过 Agent 从监控对象收集数据，存储在自身数据库中，Grafana 从 Zabbix 查询数据并进行可视化展示。
监控中数据获取有主动监控和被动监控两种方式：
主动监控是 Zabbix Agent 主动向 Zabbix Server 发送数据，监控对象需要知道 Zabbix Server 的地址和端口。好处是能减轻 Zabbix Server 的负载，适合大规模监控场景；坏处是监控对象配置相对复杂，且 Zabbix Server 不易发现监控对象离线；麻烦是每个监控对象都要配置 Zabbix Server 信息。
被动监控是 Zabbix Server 主动向 Zabbix Agent 请求数据，Zabbix Server 需要知道监控对象的地址和端口，监控对象开放端口供 Zabbix Server 访问。好处是监控对象配置简单，Zabbix Server 容易发现监控对象离线，便于集中管理；坏处是当监控对象较多时，会增加 Zabbix Server 的负载；麻烦是 Zabbix Server 需要维护监控对象列表。
监控对象（客户端）
Zabbix Agent 是 Zabbix 官方提供的客户端程序，用于收集监控对象的各类数据。
Linux 系统安装 Zabbix Agent：
```
# 安装 Zabbix 源
rpm -Uvh https://repo.zabbix.com/zabbix/6.0/rhel/7/x86_64/zabbix-release-6.0-2.el7.noarch.rpm
yum clean all
```
# 安装 Zabbix Agent
`yum install -y zabbix-agent`
# 修改配置文件
`vi /etc/zabbix/zabbix_agentd.conf`
# 配置 Zabbix Server 地址
`Server=192.168.1.100`
# 配置主动模式下 Zabbix Server 地址
`ServerActive=192.168.1.100`
# 配置主机名（需与 Zabbix Server 中配置一致）
`Hostname=Linux-Server-01`
# 启动并设置开机自启
```
systemctl start zabbix-agent
systemctl enable zabbix-agent
```

Windows 系统安装 Zabbix Agent：
从 Zabbix 官网下载对应版本的 Zabbix Agent 安装包，运行安装程序，按照提示进行安装。安装完成后，修改配置文件 zabbix_agentd.conf，配置 Server、ServerActive 和 Hostname 等参数，然后启动 Zabbix Agent 服务，并设置为开机自启。
监控系统（服务端）
先安装 Docker 和 Docker Compose V2，然后通过容器部署 Zabbix Server、Zabbix Database 和 Grafana。为简单起见，暂不考虑复杂的安全配置和服务发现机制。
创建相关目录并设置权限：
```
mkdir zabbix-server
mkdir zabbix-db
mkdir grafana
sudo chown 1000:1000 zabbix-server/
sudo chown 999:999 zabbix-db/
sudo chown 472:0 grafana/
```

编辑 docker-compose.yml：
```
services:
  zabbix-db:
    image: mysql:8.0
    container_name: zabbix-db
    restart: always
    volumes:
      - ./zabbix-db:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=zabbixroot
      - MYSQL_DATABASE=zabbix
      - MYSQL_USER=zabbix
      - MYSQL_PASSWORD=zabbixpass
    command: --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci

  zabbix-server:
    image: zabbix/zabbix-server-mysql:6.0
    container_name: zabbix-server
    restart: always
    depends_on:
      - zabbix-db
    volumes:
      - ./zabbix-server:/usr/lib/zabbix/alertscripts
    environment:
      - DB_SERVER_HOST=zabbix-db
      - MYSQL_DATABASE=zabbix
      - MYSQL_USER=zabbix
      - MYSQL_PASSWORD=zabbixpass
      - MYSQL_ROOT_PASSWORD=zabbixroot
    ports:
      - 10051:10051

  grafana:
    image: grafana/grafana
    container_name: grafana
    restart: always
    ports:
      - 3000:3000
    volumes:
      - ./grafana:/var/lib/grafana
    environment:
      - GF_INSTALL_PLUGINS=alexanderzobnin-zabbix-app
      - GF_SECURITY_ADMIN_PASSWORD=zabbix123
      - GF_SERVER_ENABLE_GZIP=true
    depends_on:
      - zabbix-server
```

启动容器：
`docker compose up -d`

配置 Zabbix Server
访问 Zabbix Web 界面（默认地址为 http:// 服务器 IP/zabbix），使用默认用户名 Admin 和密码 zabbix 登录。
进入 “配置”->“主机”，点击 “创建主机”，填写主机名称（与 Agent 配置中的 Hostname 一致）、可见名称和 IP 地址等信息，然后在 “模板” 选项卡中关联合适的模板（如 Template OS Linux），点击 “添加” 完成主机添加。
配置 Grafana
用 admin 和默认密码（GF_SECURITY_ADMIN_PASSWORD）登录 Grafana（默认地址为 http:// 服务器 IP:3000）。
进入 “插件”，找到已安装的 Zabbix 插件并启用。
进入 “配置”->“数据源”，点击 “添加数据源”，选择 Zabbix，配置 Zabbix 服务器[URL](http://zabbix-server/zabbix/api_jsonrpc.php)，输入 Zabbix 的用户名和密码，点击 “保存 & 测试”。
进入 “仪表盘”->“导入”，输入 Zabbix 相关的仪表盘 ID（如 8321），选择已配置的 Zabbix 数据源，完成仪表盘导入。
至此，一个简单的 Zabbix+Grafana 监控可视化平台就搭建完成了。在生产环境中，还需要考虑数据库备份、安全加固（如启用 HTTPS、设置复杂密码等）、Zabbix 代理的部署以分担 Server 压力等。