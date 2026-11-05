---
title: Alertmanager
date: 2025-04-02 13:48:00 +0800
categories: [三方插件, 监控与报警]  # categories是文章的分类，支持至多两级分类
tags: [Prometheus, Alertmanager]  # tags是文章的标签，可以有任意多个
---

# 一、核心功能与架构  

Alertmanager 是 Prometheus 生态的告警管理组件，负责处理 Prometheus Server 发出的警报，实现去重、分组、路由和多渠道通知。  

核心能力：  

1. 告警去重：合并重复警报（基于相同标签集）。  

2. 分组聚合：将相关告警合并为单个通知（如同一服务的多个实例故障）。  

3. 静默与抑制：临时屏蔽非关键告警或依赖链告警（如主机宕机后无需重复通知其上的服务）。  

4. 多通道通知：支持邮件、Slack、Webhook、PagerDuty 等 10\+ 通知集成。  



---



# 二、安装与基础配置  

## 快速部署  

- Docker 运行：  

    ```Plain Text
    docker run -d --name alertmanager -p 9093:9093 \  
      -v /path/to/alertmanager.yml:/etc/alertmanager/alertmanager.yml \  
      quay.io/prometheus/alertmanager:latest  
    ```

- 二进制包安装：  

    ```Plain Text
    wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz  
    tar xvfz alertmanager-*.tar.gz  
    ./alertmanager --config.file=alertmanager.yml  
    ```



## 配置文件结构（`alertmanager.yml`）  

```Plain Text
global:  
  resolve_timeout: 5m        # 告警恢复确认超时  
  smtp_smarthost: 'smtp.example.com:465'  
  smtp_from: 'alert@example.com'  
  smtp_auth_username: 'user'  
  smtp_auth_password: 'pass'  
 
route:                       # 告警路由树根节点  
  group_by: [alertname, cluster]  
  group_wait: 30s            # 初次发送等待时间（聚合同组告警）  
  group_interval: 5m         # 同一组新告警发送间隔  
  repeat_interval: 12h       # 重复发送未恢复告警的间隔  
  receiver: 'default-email'  # 默认接收器  
 
receivers:  
- name: 'default-email'  
  email_configs:  
  - to: 'ops-team@example.com'  
    send_resolved: true      # 是否通知告警恢复  
```



---



# 三、告警路由与高级策略  

## 路由树（Route）配置示例  

```Plain Text
route:  
  routes:  
  - match:                   # 匹配特定标签  
      severity: critical  
      team: database  
    receiver: 'db-pager'  
    continue: false           # 匹配后停止后续路由  
  - match_re:                # 正则匹配（如环境标签）  
      env: (prod|staging)  
    receiver: 'prod-slack'  
  - receiver: 'fallback-sms' # 默认路由  
```



## 告警抑制（Inhibition Rules）  

避免冗余通知（如网络故障时抑制依赖服务告警）：  

```Plain Text
inhibit_rules:  
- source_match:  
    severity: 'critical'  
    alertname: 'NetworkDown'  
  target_match:  
    severity: 'warning'  
  equal: ['region']          # 仅在相同 region 时抑制  
```



## 静默（Silences）配置  

通过 UI 或 API 临时屏蔽告警：  

- Web UI：访问 `http://localhost:9093` → "Silences" → 创建静默规则。  

- API 调用：  

    ```Plain Text
    curl -X POST -H "Content-Type: application/json" \  
      -d '{"matchers":[{"name":"alertname","value":"HighLatency"}]}' \  
      http://localhost:9093/api/v2/silences  
    ```



---



# 四、通知渠道集成  

## 支持的主流接收器  

|类型|配置示例|适用场景|
|---|---|---|
|Slack|`slack_configs` \+ Webhook URL|团队即时沟通|
|Webhook|自定义 HTTP 接口（如钉钉、企业微信）|内部系统集成|
|PagerDuty|`pagerduty_configs` \+ Integration Key|值班调度与自动升级|
|SMS|通过 Twilio 或第三方网关 API|紧急事件通知|



## Webhook 示例（对接企业微信机器人）  

```Plain Text
receivers:  
- name: 'wechat-webhook'  
  webhook_configs:  
  - url: 'https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=xxx'  
    send_resolved: true  
```



---



# 五、最佳实践与故障排查  

## 告警分级策略  

- 紧急（Critical）：影响核心业务（如支付失败），通知方式：电话 \+ SMS。  

- 警告（Warning）：潜在风险（如磁盘使用率80%），通知方式：Slack。  

- 信息（Info）：无需立即处理（如备份完成），仅记录日志。  



## 关键配置检查项  

- 验证配置文件语法：`amtool check-config alertmanager.yml`  

- 测试路由规则：`amtool config routes test --config.file=alertmanager.yml --verify.receivers=db-pager`  



## 常见问题解决  

|问题现象|排查步骤|
|---|---|
|未收到通知|检查接收器配置、网络连通性、Prometheus 告警规则状态|
|重复告警|调整 `group_wait` 和 `group_interval`|
|Webhook 超时|增加 `webhook_configs` 中的 `send_timeout` 参数|



---



# 六、与 Prometheus 集成  

## Prometheus 告警规则配置：  

```Plain Text
# prometheus.yml  
rule_files:  
  - /etc/prometheus/alerts.yml  
alerting:  
  alertmanagers:  
    - static_configs:  
        - targets: ['alertmanager:9093']  
```

## 告警规则示例（`alerts.yml`）：  

```Plain Text
groups:  
- name: node-alerts  
  rules:  
  - alert: HighMemoryUsage  
    expr: (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 < 10  
    for: 5m  
    labels:  
      severity: critical  
    annotations:  
      summary: "节点内存不足 ({{ $value }}% 可用)"  
```



---



# 附录  

- 官方文档：  

- 社区模板库：  

- 监控大屏模板：Grafana Dashboard ID `11074`（Alertmanager 状态总览）

