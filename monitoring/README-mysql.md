# MySQL 监控体系接入指南 (独立服务架构)

本文档旨在指导如何在 **MySQL 服务** 与 **监控体系 (Prometheus + Grafana)** 分别位于两个独立 `docker-compose.yml` 文件的情况下，安全、正确地接入 MySQL 监控。

## 核心原则
1.  **服务隔离**：不合并两个 `docker-compose.yml` 文件，保持架构独立。
2.  **网络互通**：通过 Docker **外部共享网络 (External Network)** 实现服务间通信，避免将 MySQL 端口暴露到公网。
3.  **配置安全**：使用专用的监控数据库账号，并通过配置文件 (`.my.cnf`) 而非环境变量传递敏感信息，避免转义错误。

---

## 前置准备

- 已运行独立的 MySQL 容器服务。
- 已运行独立的监控栈 (Prometheus + Grafana + Exporters)。
- 宿主机已安装 Docker 及 Docker Compose。

---

## 步骤 1：创建共享网络

为了让两个独立的 Compose 项目能够互相访问，需要创建一个外部网络。

```bash
docker network create monitoring-bridge
```

---

## 步骤 2：配置 MySQL 服务

### 2.1 创建监控专用账号

登录 MySQL 容器，创建仅具备监控权限的账号（避免使用 root）。

```bash
docker exec -it mysql8 mysql -u root -p
```

执行以下 SQL：

```sql
-- 创建用户 (请修改 exporter_password 为强密码)
CREATE USER 'exporter'@'%' IDENTIFIED BY 'exporter_password' WITH MAX_USER_CONNECTIONS 3;

-- 授权最小权限集
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'%';

FLUSH PRIVILEGES;
EXIT;
```

### 2.2 修改 MySQL 的 `docker-compose.yml`

在 MySQL 的 compose 文件中，将刚才创建的共享网络加入配置。

```yml
services:
  mysql:
    # ... 其他配置保持不变 ...
    networks:
      - mysql_net
      - monitoring-bridge  # <--- 新增：加入共享网络
  
# ... 其他配置 ...

networks:
  mysql_net:
    driver: bridge
  monitoring-bridge:     # <--- 新增：声明为外部网络
    external: true
```

应用更改：
```bash
docker-compose up -d
```

---

## 步骤 3：配置监控体系 (Prometheus Stack)

### 3.1 创建 Exporter 配置文件

在监控体系的目录下（例如 `./prometheus/`），创建 `.my.cnf` 文件。使用配置文件比环境变量更稳定，可避免密码特殊字符导致的解析错误。

```bash
mkdir -p ./prometheus
nano ./prometheus/.my.cnf
```

**文件内容**：
```ini
[client]
user=exporter
password=exporter_password  # 替换为步骤 2.1 设置的密码
host=mysql                  # 使用 MySQL 的服务名，而非 IP
port=3306
```

### 3.2 修改监控体系的 `docker-compose.yml`

找到 `mysqld-exporter` 服务，进行以下修改：
1.  挂载 `.my.cnf` 文件。
2.  指定配置文件启动参数。
3.  移除 `DATA_SOURCE_NAME` 环境变量。
4.  加入共享网络。

```yml
services:
  mysqld-exporter:
    image: prom/mysqld-exporter:v0.15.0
    container_name: mysqld-exporter
    restart: unless-stopped
    ports:
      - "9104:9104"
    volumes:
      # 挂载配置文件
      - ./prometheus/.my.cnf:/etc/mysqld_exporter/.my.cnf:ro
    command:
      # 指定配置文件路径
      - '--config.my-cnf=/etc/mysqld_exporter/.my.cnf'
    networks:
      - monitoring
      - monitoring-bridge  # <--- 新增：加入共享网络以访问 MySQL
    depends_on:
      - prometheus

networks:
  monitoring:
    driver: bridge
  monitoring-bridge:     # <--- 新增：声明为外部网络
    external: true
```

应用更改：
```bash
docker-compose up -d mysqld-exporter
```

---

## 步骤 4：配置 Prometheus 抓取

确保 `./prometheus/prometheus.yml` 中包含对 exporter 的抓取配置。

```yaml
scrape_configs:
  # ... 其他配置 ...

  - job_name: 'mysql'
    static_configs:
      - targets: ['mysqld-exporter:9104']  # 使用服务名访问
```

重启 Prometheus 使配置生效：
```bash
docker restart prometheus
```

---

## 步骤 5：配置 Grafana 面板

1.  登录 Grafana (默认 `http://localhost:3000`)。
2.  确认 **Data Sources** 中已添加 Prometheus (URL: `http://prometheus:9090`)。
3.  点击 **Dashboards** -> **Import**。
4.  输入社区面板 ID：**7362** (Percona MySQL Overview) 或 **11323**。
5.  加载后选择 Prometheus 数据源，点击 **Import**。

---

## 验证与排查

### 1. 验证连通性
查看 `mysqld-exporter` 日志，应显示 `Listening on :9104` 且无报错：
```bash
docker logs -f mysqld-exporter
```

### 2. 验证指标抓取
访问 `http://localhost:9104/metrics`，应能看到 `mysql_global_status...` 等指标。
访问 `http://localhost:9090/targets`，`mysql` 任务状态应为 **UP**。

### 3. 常见错误排查

| 错误现象                       | 可能原因                    | 解决方案                                            |
| :----------------------------- | :-------------------------- | :-------------------------------------------------- |
| `no user specified in section` | 未挂载 `.my.cnf` 或路径错误 | 检查 volumes 挂载路径及 command 参数                |
| `connection refused`           | 网络不通                    | 确认 MySQL 和 Exporter 都加入了 `monitoring-bridge` |
| `Access denied`                | 密码错误或权限不足          | 检查 `.my.cnf` 密码，确认 MySQL 用户权限            |
| `unknown host`                 | 无法解析 MySQL 服务名       | 确认 `.my.cnf` 中 host 填写的是 `mysql` (服务名)    |

---

## 总结

通过上述步骤，我们在保持两个服务独立部署的前提下，利用 Docker 外部网络实现了安全通信，并通过配置文件方式解决了 exporter 的认证问题。此方案既保证了架构的解耦，又确保了监控数据的稳定采集。