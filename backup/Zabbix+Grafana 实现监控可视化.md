Grafana 是一款专业的数据可视化展示平台，能对接包括 Zabbix 在内的多种数据源；Zabbix 则是企业级开源监控解决方案，集成了监控、报警及数据存储等功能。二者配合时，Zabbix 通过 Agent 从监控对象收集数据并存储在自身数据库，Grafana 从 Zabbix 中查询数据并进行可视化呈现。


监控中数据获取分为主动监控和被动监控两种模式：  
- **主动监控**：Zabbix Agent 主动向 Zabbix Server 发送数据，监控对象需知晓 Zabbix Server 的地址与端口。优势在于可减轻 Zabbix Server 的负载，适合大规模监控场景；但监控对象配置相对复杂，且 Zabbix Server 较难发现监控对象离线，需为每个监控对象单独配置 Zabbix Server 信息。  
- **被动监控**：Zabbix Server 主动向 Zabbix Agent 请求数据，Zabbix Server 需掌握监控对象的地址与端口，且监控对象需开放端口供 Zabbix Server 访问。优势是监控对象配置简单，Zabbix Server 易发现监控对象离线，便于集中管理；但监控对象较多时会增加 Zabbix Server 负载，且需由 Zabbix Server 维护监控对象列表。  


监控对象（客户端）需部署 Zabbix Agent，这是 Zabbix 官方提供的客户端程序，用于收集监控对象的各类数据。


# Linux 系统安装 Zabbix Agent 步骤  
## 安装 Zabbix 源  
```bash
rpm -Uvh https://repo.zabbix.com/zabbix/6.0/rhel/7/x86_64/zabbix-release-6.0-2.el7.noarch.rpm  # 根据系统版本调整，此处以 RHEL 7 为例
yum clean all
```  

## 安装 Zabbix Agent  
```bash
yum install -y zabbix-agent
```  

## 修改配置文件  
```bash
vi /etc/zabbix/zabbix_agentd.conf 
```  

## 配置核心参数  
- 被动模式下 Zabbix Server 地址：`Server=192.168.1.100`（允许该地址的 Server 拉取数据）  
- 主动模式下 Zabbix Server 地址：`ServerActive=192.168.1.100`（Agent 主动推送数据的目标地址）  
- 主机名（需与 Zabbix Server 中配置完全一致）：`Hostname=Linux-Server-01`  


## 启动并设置开机自启  
```bash
systemctl start zabbix-agent
systemctl enable zabbix-agent  # 确保服务开机自动运行
```  


# Grafana 容器部署配置（Docker Compose 示例）  
```yaml
version: '3'
services:
  grafana:
    image: grafana/grafana
    container_name: grafana
    restart: always
    ports:
      - 3000:3000  # 映射 Grafana 端口到主机
    volumes:
      - ./grafana:/var/lib/grafana  # 持久化存储 Grafana 数据
    environment:
      - GF_INSTALL_PLUGINS=alexanderzobnin-zabbix-app  # 自动安装 Zabbix 插件
      - GF_SECURITY_ADMIN_PASSWORD=zabbix123  # 初始管理员密码，建议生产环境修改为复杂密码
      - GF_SERVER_ENABLE_GZIP=true  # 启用 GZIP 压缩提升性能
    depends_on:
      - zabbix-server  # 依赖 Zabbix Server 服务
```  


# 配置 Zabbix Server  
1. 访问 Zabbix Web 界面（默认地址：http://[服务器IP]/zabbix），使用默认账号 `Admin`、密码 `zabbix` 登录（首次登录建议立即修改密码）。  
2. 进入「配置」→「主机」，点击「创建主机」：  
   - 填写主机名称（需与 Agent 配置的 `Hostname` 完全一致）、可见名称及 IP 地址。  
   - 切换到「模板」选项卡，关联合适的监控模板（如 `Template OS Linux` 用于监控 Linux 系统）。  
3. 点击「添加」完成主机配置。  


# 配置 Grafana  
1. 登录 Grafana（默认地址：http://[服务器IP]:3000），使用账号 `admin` 和上述 `GF_SECURITY_ADMIN_PASSWORD` 配置的密码登录。  
2. 启用 Zabbix 插件：进入「插件」，找到已安装的 `Zabbix` 插件并启用。  
3. 配置数据源：  
   - 进入「配置」→「数据源」，点击「添加数据源」，选择 `Zabbix`。  
   - 配置 Zabbix 服务器 URL：`http://zabbix-server/zabbix/api_jsonrpc.php`（若 Zabbix 与 Grafana 不在同一容器网络，需替换为实际 IP）。  
   - 输入 Zabbix 的登录用户名和密码，点击「保存 & 测试」确认连接正常。  
4. 导入仪表盘：  
   - 进入「仪表盘」→「导入」，输入 Zabbix 相关仪表盘 ID（如 `8321`，可在 Grafana 官方仪表盘库查询更多）。  
   - 选择已配置的 Zabbix 数据源，完成导入。  


至此，基础的 Zabbix+Grafana 监控可视化平台搭建完成。生产环境中，还需补充：数据库定期备份、安全加固（如启用 HTTPS、设置复杂密码、限制访问 IP）、Zabbix Proxy 部署以分担 Server 压力，以及监控指标精细化配置（如自定义监控项、调整采集频率）等。