---
type: essay
created: 2025-12-16
tags: []
---

# cuda开发环境搭建

## 起因

## 思考过程

## 我的结论


让docker走clash代理

```bash
# 1. 创建配置文件夹
sudo mkdir -p /etc/systemd/system/docker.service.d

# 2. 创建并编辑配置文件
sudo nano /etc/systemd/system/docker.service.d/http-proxy.conf
```
在打开的编辑器中，**完整复制并粘贴**以下内容：
```ini
[Service] Environment="HTTP_PROXY=http://127.0.0.1:7890" Environment="HTTPS_PROXY=http://127.0.0.1:7890" Environment="NO_PROXY=localhost,127.0.0.1"
```
(按 `Ctrl+O` 保存，`Enter` 确认，`Ctrl+X` 退出)
```bash
sudo systemctl daemon-reload && sudo systemctl restart docker
```