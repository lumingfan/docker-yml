


这是一份为你量身定制的 `README.md`，将我们上述的整个架构改造、配置过程以及排坑经验进行了完整的总结。你可以直接将它保存到你的项目根目录中。

---

# Docker Compose 架构集成 Canal 与监控指南

本项目基于现有的多目录 Docker Compose 架构（包含 `mysql`, `es`, `monitoring`），新增了 **Canal** 服务以实现 MySQL Binlog 监听，并将其 Metrics 接入了现有的 Prometheus + Grafana 监控体系。

## 📁 目录结构演进

引入 Canal 后的完整项目架构如下：

```text
.
├── canal                       <-- [新增] Canal 服务模块
│   ├── conf
│   │   ├── canal.properties    <-- Canal 主配置 (需包含完整默认配置并开启监控)
│   │   └── instance.properties <-- Canal 实例配置 (定义如何连接 MySQL)
│   ├── docker-compose.yml      <-- Canal 容器编排
│   └── logs                    <-- Canal 日志挂载目录
├── es                          <-- 现有 Elasticsearch 服务
├── monitoring                  <-- 现有监控体系 (Prometheus + Grafana)
│   ├── prometheus
│   │   └── prometheus.yml      <-- 已新增对 Canal metrics 的抓取
│   └── ...
└── mysql                       <-- 现有 MySQL 服务
    ├── conf
    │   └── my.cnf              <-- 已开启 Binlog
    ├── init-scripts
    │   └── init_canal.sql      <-- 已新增 Canal 授权账号
    └── ...
```

---

## 🛠️ 前置准备：全局网络设置

---

## 🚀 步骤一：配置 MySQL 

1. **初始化同步账号**：
   在 `mysql/init-scripts/` 下新建 SQL 脚本，MySQL 启动时自动创建账号：
   ```sql
   CREATE USER 'canal'@'%' IDENTIFIED BY 'canal';
   GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
   FLUSH PRIVILEGES;
   ```
   

---

## 🚀 步骤二：配置 Canal

### 1. 准备配置文件

**重要提示：主配置文件 `canal.properties` 不能只写增量配置，必须基于官方完整配置进行修改，否则容器将无法启动 11112 端口！**

```bash
https://github.com/alibaba/canal/blob/master/deployer/src/main/resources/canal.properties
```

**修改 `canal/conf/canal.properties`**：
找到 `canal.metrics.pull.port` 所在行，取消注释并设置端口：
```properties
canal.metrics.pull.port=11112
```

**创建 `canal/conf/instance.properties`**：
配置 Canal 连接到 MySQL：
```properties
canal.instance.mysql.slaveId=1234
canal.instance.master.address=mysql8:3306  # 注意这里的 mysql 是容器名
canal.instance.dbUsername=canal
canal.instance.dbPassword=canal
canal.instance.connectionCharset=UTF-8
canal.instance.filter.regex=.*\\..*       # 监听所有库表
```

### 2. Docker Compose 编排

创建 `canal/docker-compose.yml`：
```yaml
version: '3.8'

services:
  canal-server:
    image: canal/canal-server:v1.1.7
    container_name: canal-server
    ports:
      - "11111:11111" # 客户端直连端口
      - "11112:11112" # Prometheus metrics 端口
    volumes:
      - ./conf/instance.properties:/home/admin/canal-server/conf/example/instance.properties
      - ./conf/canal.properties:/home/admin/canal-server/conf/canal.properties
      - ./logs:/home/admin/canal-server/logs
    networks:
      - monitoring-bridge
    restart: always

networks:
  monitoring-bridge:
    external: true
```

启动 Canal：
```bash
cd canal
docker-compose up -d
```

---

## 🚀 步骤三：配置 Prometheus 监控与 Grafana

1. **修改 Prometheus 配置**：
   在 `monitoring/prometheus/prometheus.yml` 中新增 Canal 的拉取任务。
   **注意 YAML 语法，`targets:` 后面必须有空格！**

   ```yaml
   scrape_configs:
     - job_name: 'canal'
       scrape_interval: 15s
       static_configs:
         # 注意冒号后面的空格，否则会报 yaml: unmarshal errors
         - targets: ['canal-server:11112'] 
   ```

2. **重启 Prometheus**：
   
   ```bash
   cd monitoring
   docker-compose restart prometheus
   ```
   
3. **导入 Grafana 看板**：
   - 登录 Grafana -> 左侧点击 "+" -> Import。
   - 填入官方模板 JSON 获取地址：[`Canal_instances_grafana.json`](https://raw.githubusercontent.com/alibaba/canal/master/prometheus/Canal_instances_grafana.json)。
     - https://raw.githubusercontent.com/alibaba/canal/master/deployer/src/main/resources/metrics/Canal_instances_tmpl.json
   - 选择对应 Prometheus 数据源即可查看 Canal 的 QPS、延迟等监控图表。

---

## 🛑 常见踩坑与排错记录 (Troubleshooting)

### 1. Prometheus 报错 `yaml: unmarshal errors` 
**错误日志**：`cannot unmarshal !!str targets... into struct`
**原因**：YAML 语法要求键值对的冒号后必须带有空格。
**解决**：将 `- targets:['canal-server:11112']` 修正为 `- targets:['canal-server:11112']`。

### 2. Prometheus 报错 `connection refused` 
**错误日志**：`Get "http://canal-server:11112/metrics": dial tcp... connect: connection refused`
**原因**：Prometheus 成功解析了 IP，但 Canal 容器内部并没有监听 11112 端口。通常是因为自己手写的 `canal.properties` 不完整，导致 Canal 核心服务启动异常或 Metrics 模块未成功初始化。
**解决**：
1. 检查 `docker logs canal-server` 以及 `canal/logs/example/example.log` 看是否有 MySQL 连接报错。
2. 确保 `canal.properties` 是下载的官方 **完整默认配置** 并只修改了 `canal.metrics.pull.port=11112`，而不是自己新建的只有两三行配置的文件。
