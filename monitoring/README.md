---

# 📊 Prometheus + Grafana 监控栈部署指南

本项目使用 Docker Compose 快速部署一套完整的监控解决方案，包含 **Prometheus**（指标采集与存储）、**Grafana**（数据可视化）和 **Node Exporter**（主机指标采集）。


## ✨ 功能特性

- **自动化部署**：一键启动所有组件。
- **数据持久化**：监控数据和本地配置挂载到本地磁盘，重启不丢失。
- **自动数据源配置**：Grafana 自动连接 Prometheus，无需手动配置。
- **主机监控**：内置 Node Exporter，自动采集 CPU、内存、磁盘、网络等指标。
- **易于扩展**：支持轻松添加新的监控目标。

---

## 📋 前置要求

- **Docker**: 版本 20.10+
- **Docker Compose**: 版本 2.0+
- **端口**: 确保服务器端口 `3000`, `9090`, `9100` 未被占用。
- **权限**: 需要 `sudo` 或当前用户有 Docker 权限。

---

## 📂 项目结构

```text
monitoring/
├── docker-compose.yml          # 核心编排文件
├── prometheus/
│   └── prometheus.yml          # Prometheus 配置文件
├── grafana/
│   └── provisioning/           # Grafana 自动配置
│       ├── datasources/        # 数据源配置
│       └── dashboards/         # 仪表板配置
├── data/                       # 数据持久化目录 (自动创建)
│   ├── prometheus/
│   └── grafana/
└── README.md                   # 本说明文档
```

---

## 🚀 快速启动

1. **克隆或下载项目**
   确保所有文件位于同一目录下。

2. **修复权限 (Linux/Mac 重要)**
   为避免容器启动时报 `Permission denied`，请执行：
   ```bash
   mkdir -p data/prometheus data/grafana
   sudo chown -R 472:472 data/grafana      # Grafana 用户 ID
   sudo chown -R 65534:65534 data/prometheus # Prometheus 用户 ID
   ```

3. **启动服务**
   ```bash
   docker-compose up -d
   ```

4. **验证状态**
   ```bash
   docker-compose ps
   ```
   确保所有容器状态为 `Up`。

---

## 🔐 访问信息

| 组件 | 地址 | 默认账号 | 默认密码 | 说明 |
| :--- | :--- | :--- | :--- | :--- |
| **Grafana** | http://localhost:3000 | `admin` | `admin123` | 可视化面板 |
| **Prometheus** | http://localhost:9090 | 无 | 无 | 指标查询与管理 |
| **Node Exporter** | http://localhost:9100 | 无 | 无 | 裸机指标接口 |

> ⚠️ **警告**：首次登录后，请务必在 Grafana 中修改默认密码！

---

## 🔍 Prometheus 使用指南

Prometheus 是监控的核心，负责抓取和存储数据。

### 1. 检查监控目标 (Targets)
访问 `http://localhost:9090/targets`。
- 所有状态应显示为 **UP** (绿色)。
- 如果显示 **DOWN** (红色)，请检查 `docker-compose logs prometheus` 排查网络或配置问题。

### 2. 使用 Graph 进行查询 (PromQL)
访问 `http://localhost:9090/graph`，在表达式框中输入以下示例：

| 描述 | PromQL 表达式 |
| :--- | :--- |
| **检查目标是否存活** | `up` |
| **CPU 使用率 (平均值)** | `100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)` |
| **内存使用率** | `100 - ((node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100)` |
| **磁盘使用率** | `100 - ((node_filesystem_avail_bytes / node_filesystem_size_bytes) * 100)` |
| **网络接收流量** | `rate(node_network_receive_bytes_total[5m])` |

### 3. 查看配置
访问 `http://localhost:9090/config` 可查看当前加载的 `prometheus.yml` 内容，确认修改是否生效。

---

## 📈 Grafana 使用指南

Grafana 用于将 Prometheus 的数据转化为漂亮的图表。

### 1. 登录与数据源验证
1. 浏览器访问 `http://localhost:3000`。
2. 使用默认账号登录。
3. 进入 **Configuration (齿轮图标) -> Data Sources**。
4. 确认 **Prometheus** 列表中存在且状态为 `Health: OK`。
   *注：本项目已通过 `provisioning` 自动配置，通常无需手动操作。*

### 2. 导入现成仪表板 (推荐)
不要从零开始，使用社区成熟的模板：
1. 点击左侧菜单 **Dashboards -> Import**。
2. 输入社区 URL：**`https://grafana.com/grafana/dashboards/1860-node-exporter-full/`** (Node Exporter Full)。
3. 点击 **Load**，选择 Prometheus 数据源。
4. 点击 **Import**。
5. 你将看到一个包含主机所有详细指标的专业仪表盘。

### 3. 创建自定义面板
1. 点击 **Create -> Dashboard -> Add new panel**。
2. 在下方查询框输入 PromQL (参考上文 Prometheus 指南)。
3. 在右侧设置标题、可视化类型 (Graph, Gauge, Stat 等)。
4. 点击 **Apply** 保存。
5. 点击右上角 **Save dashboard** 命名并保存。

---

## ⚙️ 配置与扩展

### 1. 添加新的监控目标
若要监控其他服务（如 MySQL, Nginx），需两步走：

**Step 1: 修改 `prometheus/prometheus.yml`**
```yaml
scrape_configs:
  # ... 现有配置 ...
  - job_name: 'nginx'
    static_configs:
      - targets: ['nginx-exporter:9113']
```

**Step 2: 重启 Prometheus**
```bash
docker-compose restart prometheus
```
*注：由于配置了 `--web.enable-lifecycle`，也可通过 API 重载配置：*
```bash
curl -X POST http://localhost:9090/-/reload
```

### 2. 修改数据保留时间
默认保留 15 天。修改 `docker-compose.yml` 中 Prometheus 的 command：
```yaml
command:
  - '--storage.tsdb.retention.time=30d'  # 改为 30 天
```
修改后需重启容器：`docker-compose up -d`。

### 3. 修改 Grafana 密码
不要直接改配置文件，建议在登录后通过 UI 修改，或修改 `docker-compose.yml` 环境变量后重建容器：
```yaml
environment:
  - GF_SECURITY_ADMIN_PASSWORD=你的强密码
```

---

## 🛡️ 安全建议 (生产环境必读)

1. **修改默认密码**：Grafana 默认密码 `admin123` 极其危险。
2. **限制网络访问**：
   - 生产环境不要将 `9090` 和 `3000` 绑定到 `0.0.0.0`。
   - 建议使用 Nginx 反向代理 + HTTPS。
   - 或者在 `docker-compose.yml` 中只监听本地：`127.0.0.1:3000:3000`。
3. **开启认证**：Prometheus 原生无认证，务必通过 Nginx 或 Auth 中间件保护。
4. **防火墙**：仅允许受信任的 IP 访问监控端口。

---

## 🔧 运维与维护

### 1. 查看日志
```bash
# 查看所有日志
docker-compose logs -f

# 查看特定服务日志
docker-compose logs -f grafana
```

### 2. 备份数据
数据存储在 `./data` 目录，直接打包即可备份：
```bash
tar -czvf monitoring-backup-$(date +%F).tar.gz ./data
```

### 3. 更新版本
1. 修改 `docker-compose.yml` 中的镜像版本号。
2. 拉取新镜像并重启：
   ```bash
   docker-compose pull
   docker-compose up -d
   ```

### 4. 清理资源
```bash
# 停止并删除容器 (数据保留)
docker-compose down

# 停止并删除容器及数据卷 (数据丢失！慎用)
docker-compose down -v
```

---

## ❓ 常见问题 (FAQ)

**Q1: Grafana 启动失败，日志显示 `mkdir: permission denied`**
- **A**: 这是本地目录权限问题。请执行“快速启动”中的 `chown` 命令，确保宿主机目录所有者与容器内用户 ID 匹配。

**Q2: Prometheus  Targets 显示 DOWN**
- **A**: 
  1. 检查容器网络：`docker network inspect monitoring_default`。
  2. 检查目标服务是否启动。
  3. 检查 `prometheus.yml` 中的地址是否填写了容器名（如 `node-exporter`）而非 `localhost`。

**Q3: 时间不对，图表数据有偏差**
- **A**: 确保宿主机和容器的时间同步。检查时区设置，可在 `docker-compose.yml` 中添加 `- /etc/localtime:/etc/localtime:ro` 挂载时间文件。

**Q4: 内存占用过高**
- **A**: Prometheus 是内存敏感型应用。
  1. 减少 `scrape_interval` (默认 15s 可改为 30s 或 60s)。
  2. 缩短 `retention.time`。
  3. 使用 `drop_labels` 过滤不需要的指标。

---

## 📚 参考资源

- [Prometheus 官方文档](https://prometheus.io/docs/)
- [Grafana 官方文档](https://grafana.com/docs/)
- [PromQL 查询教程](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Grafana 仪表板市场](https://grafana.com/grafana/dashboards/)

---

**🎉 部署成功！现在你可以开始构建你的监控体系了。**
