---
title: Node Exporter
date: 2025-04-03 13:48:00 +0800
categories: [三方插件, 监控与报警]  # categories是文章的分类，支持至多两级分类
tags: [Prometheus, Node Exporter]  # tags是文章的标签，可以有任意多个
---

# 一、简介与核心功能  

Node Exporter 是 Prometheus 生态中用于采集主机级监控指标的开源工具，支持 Linux/Unix 系统，通过暴露 `/metrics` 端点提供硬件和操作系统层面的数据。  

核心监控范围：  

- CPU：使用率、负载、中断次数  

- 内存：总量、可用量、Swap 使用  

- 磁盘：I/O 吞吐、空间占用、inode 状态  

- 网络：流量统计、TCP/UDP 连接数  

- 系统：启动时间、运行进程数、文件句柄数  



---



# 二、安装与启动  

## 基础安装方式  

- 二进制包（Linux）：  

    ```Plain Text
    wget https://github.com/prometheus/node_exporter/releases/download/v1.8.0/node_exporter-1.8.0.linux-amd64.tar.gz  
    tar xvfz node_exporter-*.tar.gz  
    ./node_exporter --web.listen-address=":9100"  
    ```

- Docker 部署：  

    ```Plain Text
    docker run -d --name node_exporter -p 9100:9100 \  
      -v "/proc:/host/proc" -v "/sys:/host/sys" \  
      -v "/:/rootfs" --net="host" \  
      prom/node-exporter:latest \  
      --path.procfs=/host/proc --path.sysfs=/host/sys  
    ```

- Kubernetes DaemonSet：  

    ```Plain Text
    # 示例配置片段  
    spec:  
      containers:  
      - name: node-exporter  
        image: prom/node-exporter:latest  
        ports:  
          - containerPort: 9100  
        args:  
          - --collector.filesystem.ignored-mount-points="^/(sys|proc|dev|host|etc)($$|/)"  
    ```



## 验证运行状态  

访问 `http://<主机IP>:9100/metrics`，若返回大量 `node_` 前缀指标（如 `node_cpu_seconds_total`），则部署成功。



---



# 三、配置与高级参数  

## 常用启动参数  

|参数|作用描述|
|---|---|
|`--web.listen-address=:9100`|监听地址与端口（默认 9100）|
|`--collector.disable-defaults`|禁用所有默认收集器|
|`--collector.cpu`|单独启用 CPU 收集器|
|`--no-collector.netstat`|禁用 netstat 相关指标采集|
|`--collector.textfile.directory`|加载自定义文本指标目录（用于扩展）|



## 配置文件示例（`.yml`）  

```Plain Text
node_exporter.yml  
modules:  
  cpu:  
    enabled: true  
  memory:  
    enabled: true  
  filesystem:  
    mount-points-exclude: ["^/boot$", "^/run$"]  
```

启动命令：`./node_exporter --config.file=node_exporter.yml`  



---



# 四、与 Prometheus 集成  

## Prometheus 抓取配置  

```Plain Text
prometheus.yml  
scrape_configs:  
  - job_name: "node"  
    static_configs:  
      - targets: ["node1:9100", "node2:9100"]  
    relabel_configs:  
      - source_labels: [__address__]  
        target_label: instance  
        replacement: "主机-$1"  # 自定义实例标签  
```



## 关键监控告警规则（示例）  

```python
alerts.yml  
groups:  
- name: node-alerts  
  rules:  
  - alert: HighDiskUsage  
    expr: 100 - (node_filesystem_avail_bytes{fstype=~"ext4|xfs"} / node_filesystem_size_bytes * 100) > 90  
    for: 10m  
    labels:  
      severity: critical  
    annotations:  
      summary: "磁盘空间不足 (实例 {{ labels.instance }})"  
```



---



# 五、安全与权限管理  

## 限制访问  

- 防火墙规则：仅允许 Prometheus Server IP 访问 9100 端口  

    ```Plain Text
    ufw allow from 192.168.1.100 to any port 9100  
    ```

- TLS 加密：启用 HTTPS  

    ```Plain Text
    ./node_exporter --web.config.file=web-config.yml  
    ```

    ```Plain Text
    # web-config.yml  
    tls_server_config:  
      cert_file: node_exporter.crt  
      key_file: node_exporter.key  
    ```



## 运行权限  

- 非 root 用户运行：  

    ```Plain Text
    useradd -M -s /bin/false node_exporter  
    chown -R node_exporter:node_exporter /opt/node_exporter  
    ```



---



# 六、高级功能与扩展  

## 自定义指标扩展  

- 通过 `--collector.textfile.directory` 加载脚本生成的指标：  

    ```Plain Text
    # 示例脚本生成自定义指标  
    echo 'node_custom_temp 42' > /var/lib/node_exporter/temp.prom  
    ```



## 集成黑盒监控  

- 使用 Blackbox Exporter 联动监控主机网络可达性：  

    ```Plain Text
    # Prometheus 配置  
    - job_name: 'blackbox-node-ping'  
      metrics_path: /probe  
      params:  
        module: [icmp]  
      static_configs:  
        - targets: ['node1:9100', 'node2:9100']  
      relabel_configs:  
        - source_labels: [__address__]  
          target_label: __param_target  
        - target_label: __address__  
          replacement: blackbox-exporter:9115  
    ```



---



# 七、最佳实践与问题排查  

## 推荐配置项  

- 禁用非必要收集器：减少资源消耗（如 `--no-collector.wifi` 禁用无线网络监控）  

- 日志轮转：配置 logrotate 避免日志膨胀  

    ```Plain Text
    /var/log/node_exporter/*.log {  
      daily  
      rotate 7  
      compress  
      missingok  
    }  
    ```



## 常见问题解决  

|问题现象|解决方案|
|---|---|
|`/metrics` 无响应|检查端口冲突、进程状态、防火墙规则|
|部分磁盘未监控|调整 `--collector.filesystem.mount-points` 参数|
|权限不足导致指标缺失|以 root 临时运行或配置 Capabilities（如 `CAP_DAC_READ_SEARCH`）|



---



# 附录  

- 官方文档：  

- 监控大屏模板：Grafana Dashboard ID `1860`（Node Exporter 全指标仪表盘）  

- 性能调优：通过 `--web.max-requests=40` 限制并发请求防止过载  



\-（本指南基于 Linux x86\_64 系统环境，Windows 用户需使用 WMI Exporter 替代）\*



