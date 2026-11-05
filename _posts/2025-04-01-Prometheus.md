---
title: Prometheus
date: 2025-04-01 13:48:00 +0800
categories: [三方插件, 监控与报警]  # categories是文章的分类，支持至多两级分类
tags: [Prometheus]  # tags是文章的标签，可以有任意多个
---


# 中文文档

https://www\.prometheus\.wang/quickstart/why\-monitor\.html

# 一、简介与核心概念  

Prometheus 是一款开源的监控与警报工具，适用于云原生和分布式系统，由 SoundCloud 开发并于2016年加入 CNCF。  

核心特性：  

1. 多维数据模型：基于时间序列的键值对（指标\+标签）。  

2. 灵活查询语言（PromQL）：支持实时聚合、切片和预测分析。  

3. 主动拉取模型：通过 HTTP 抓取目标暴露的指标端点。  

4. 集成警报：与 Alertmanager 结合实现多通道通知（邮件、Slack等）。  



---



# 二、安装与部署  

## 快速安装（Linux）  

- Docker 部署：  

    ```Plain Text
    docker run -d --name prometheus -p 9090:9090 \  
      -v /path/to/prometheus.yml:/etc/prometheus/prometheus.yml \  
      prom/prometheus:latest  
    ```

- 二进制包安装：  

    ```Plain Text
    wget https://github.com/prometheus/prometheus/releases/download/v2.50.0/prometheus-2.50.0.linux-amd64.tar.gz  
    tar xvfz prometheus-*.tar.gz  
    ./prometheus --config.file=prometheus.yml  
    ```



## 基础配置（`prometheus.yml`）  

```Plain Text
global:  
  scrape_interval: 15s  # 抓取间隔  
 
scrape_configs:  
  - job_name: "node_exporter"  
    static_configs:  
      - targets: ["localhost:9100"]  # 监控本机资源  
  - job_name: "custom_app"  
    metrics_path: "/metrics"  
    static_configs:  
      - targets: ["app-server:8080"]  
```



---



# 三、核心功能与使用  

## 数据模型与指标类型  

- 指标格式：`<metric_name>{<label1>=<value1>, ...} <value> [timestamp]`  

- 四大类型：  

    - Counter：累加值（如请求总数）。  

    - Gauge：瞬时值（如内存使用量）。  

    - Histogram：分桶统计（如响应时间分布）。  

    - Summary：客户端计算的百分位数。  



## PromQL 查询示例  

- 计算 CPU 使用率：  

    ```Plain Text
    100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)  
    ```

- 检测 HTTP 错误率：  

    ```Plain Text
    sum(rate(http_requests_total{status_code=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))  
    ```

- 预测磁盘空间耗尽时间：  

    ```Plain Text
    predict_linear(node_filesystem_avail_bytes{device="/dev/sda1"}[1h], 3600*24) < 0  
    ```



## 警报规则配置（`alerts.yml`）  

```Plain Text
groups:  
- name: example  
  rules:  
  - alert: HighRequestLatency  
    expr: job:request_latency_seconds:mean5m > 0.5  
    for: 10m  
    labels:  
      severity: critical  
    annotations:  
      summary: "高请求延迟 ({{ $value }}秒)"  
      description: "实例 {{ $labels.instance }} 的延迟持续超过阈值。"  
```



---



# 四、进阶使用场景  

## 服务发现  

- 支持 Kubernetes、Consul、AWS EC2 等动态目标发现。  

- Kubernetes 示例：  

    ```Plain Text
    scrape_configs:  
      - job_name: "kubernetes-pods"  
        kubernetes_sd_configs:  
          - role: pod  
        relabel_configs:  
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]  
            action: keep  
            regex: true  
    ```



## 远程存储与长期保留  

- 集成 Thanos 或 VictoriaMetrics 实现跨集群数据聚合与长期存储。  

- 配置远程写入：  

    ```Plain Text
    remote_write:  
      - url: "http://thanos-receive:10908/api/v1/receive"  
    ```



## 联邦集群  

- 分层抓取数据，适用于大规模分布式监控：  

    ```Plain Text
    scrape_configs:  
      - job_name: "federate"  
        honor_labels: true  
        metrics_path: "/federate"  
        params:  
          match[]:  
            - '{job="prometheus"}'  
        static_configs:  
          - targets: ["prometheus-central:9090"]  
    ```



## 自定义指标导出器  

- 使用 Client Libraries（Python、Go、Java）暴露应用指标：  

    ```Plain Text
    from prometheus_client import start_http_server, Counter  
    REQUEST_COUNT = Counter('http_requests_total', 'Total HTTP Requests')  
    start_http_server(8000)  
    ```



---



# 五、最佳实践与问题排查  

## 标签管理  

- 避免标签高基数：如 `user_id` 可能导致时间序列爆炸。  

- 推荐标签：`job`、`instance`、`env`（环境标识）。  



## 性能优化  

- 内存调优：增加 `--storage.tsdb.max-block-duration` 减少内存占用。  

- 查询优化：限制范围查询的时间窗口（如 `[1h]` 而非 `[7d]`）。  



## 监控 Prometheus 自身  

- 监控指标：`prometheus_tsdb_head_samples_appended_total`（写入速率）、`prometheus_target_scrape_pool_targets`（活跃目标数）。  



## 常见问题  

- 数据丢失：检查存储目录权限和磁盘空间。  

- 查询超时：优化 PromQL 或分片查询。  



---



# 附录：工具链与生态  

1. Grafana：可视化仪表盘（推荐模板 ID：1860）。  

2. Alertmanager：多路警报路由与静默配置。  

3. Blackbox Exporter：探测 HTTP/TCP 服务可用性。  

4. Pushgateway：临时任务指标推送（如批处理作业）。  



\-（指南基于 Prometheus 2\.50 版本，适配 Kubernetes 1\.30 及更高环境）\*

