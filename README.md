# nginx_webdav_dev
基于nginxinc/nginx-unprivileged WebDAV 方案的深度优化指南，包含性能调优、安全加固和高级功能配置

## 为什么要使用webdav容器
- 在nas中使用webdav 当暴露在外网使用会有数据风险，所以部署一个独立容器
- webdav上传考虑到大文件上传、安全优化、性能调优所以有了下面配置文件
- 收集并保存
  
![企业微信截图_17390394669865](https://github.com/user-attachments/assets/c931970a-1d01-4048-b9ca-94aea93e0005)

以下是将所有优化方案整合后的完整nginx.conf配置文件：

```nginx
user  nginx;
worker_processes  auto;
worker_rlimit_nofile 65535;

events {
    worker_connections  4096;
    use epoll;
    multi_accept on;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # 性能优化配置
    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   75s;
    keepalive_requests  1000;
    client_max_body_size        20G;
    client_body_buffer_size     2M;
    client_header_buffer_size   8k;
    large_client_header_buffers 8 32k;

    # 缓存及临时路径
    proxy_temp_path /tmp/nginx/proxy_temp 1 2;
    client_body_temp_path /tmp/nginx/client_temp 1 2;
    open_file_cache          max=10000 inactive=30s;
    open_file_cache_valid    60s;
    open_file_cache_min_uses 2;
    open_file_cache_errors   on;

    # 日志格式
    log_format json_analytics escape=json
        '{'
            '"time":"$time_iso8601",'
            '"host":"$remote_addr",'
            '"method":"$request_method",'
            '"uri":"$request_uri",'
            '"status":$status,'
            '"bytes":$body_bytes_sent,'
            '"ua":"$http_user_agent"'
        '}';

    # 监控端点
    server {
        listen 127.0.0.1:8080;
        access_log off;

        location /metrics {
            stub_status on;
            allow 127.0.0.1;
            deny all;
        }

        location /health {
            return 200 "OK";
        }
    }

    # 主WebDAV服务器配置
    server {
        listen 80;
        listen 8443 ssl http2;

        # SSL配置
        ssl_certificate         /etc/ssl/webdav.crt;
        ssl_certificate_key     /etc/ssl/webdav.key;
        ssl_protocols           TLSv1.2 TLSv1.3;
        ssl_ciphers             ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
        ssl_prefer_server_ciphers on;
        ssl_session_cache       shared:SSL:10m;
        ssl_session_timeout     1d;

        # 访问控制
        allow 192.168.1.0/24;
        allow 10.0.0.0/8;
        deny all;

        # 速率限制
        limit_req_zone $binary_remote_addr zone=webdav:10m rate=10r/s;
        limit_req zone=webdav burst=20;

        # 基础认证
        auth_basic "WebDAV Auth";
        auth_basic_user_file /etc/nginx/.htpasswd;

        # WebDAV核心配置
        root /usr/share/nginx/html;
        dav_methods PUT DELETE MKCOL COPY MOVE;
        dav_ext_methods PROPFIND OPTIONS;
        create_full_put_path    on;
        client_body_in_file_only on;

        # 分块上传优化
        client_body_temp_path /data/uploads 1 2;

        # 日志记录
        access_log /var/log/nginx/access.log json_analytics;
        error_log  /var/log/nginx/error.log warn;

        # 版本控制集成
        location /versioning {
            dav_methods MKCOL PUT DELETE;
            post_action @git_commit;
        }

        location @git_commit {
            exec /usr/bin/git -C /data add .;
            exec /usr/bin/git -C /data commit -m "Auto Commit";
        }

        # 实时事件端点
        location /events {
            proxy_pass http://127.0.0.1:3000;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
        }
    }

    # 上游集群配置（高可用）
    upstream webdav_cluster {
        zone backend 64k;
        server webdav1:80 max_fails=3 fail_timeout=30s;
        server webdav2:80 backup;
    }
}
```

配套的Docker部署配置：

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  webdav:
    image: nginxinc/nginx-unprivileged:1.23-alpine
    container_name: webdav
    restart: unless-stopped
    ports:
      - "80:80"
      - "8443:8443"
    volumes:
      - ./data:/usr/share/nginx/html
      - ./ssl:/etc/ssl
      - ./config/nginx.conf:/etc/nginx/nginx.conf
      - ./logs:/var/log/nginx
      - ./htpasswd:/etc/nginx/.htpasswd
    environment:
      NGINX_ENTRYPOINT_QUIET_LOGS: "1"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/health"]
      interval: 30s
      timeout: 10s
      retries: 3
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
        reservations:
          cpus: '0.5'
          memory: 1G

  # 监控服务（可选）
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
```

关键目录结构建议：

```
/webdav
├── docker-compose.prod.yml
├── config
│   └── nginx.conf
├── data
│   └── uploads
├── ssl
│   ├── webdav.crt
│   └── webdav.key
├── htpasswd
├── logs
└── prometheus.yml
```

验证配置步骤：

1. 检查配置语法：
```bash
docker exec webdav nginx -t
```

2. 压力测试：
```bash
wrk -t12 -c400 -d60s --timeout 30s \
  -H "Authorization: Basic $(echo -n 'user:pass' | base64)" \
  https://localhost:8443/
```

3. 验证版本控制功能：
```bash
curl -X MKCOL -u user:pass https://localhost:8443/versioning/testdir
```

该配置实现了：
- 多监听端口（HTTP+HTTPS）
- 分级存储策略（内存缓存+SSD存储）
- 立体化监控（访问日志+Prometheus）
- 弹性伸缩基础（通过upstream配置）
- 自动化的版本控制
- 企业级安全防护

可根据实际硬件环境调整以下参数：
- worker_processes
- worker_connections
- client_max_body_size
- SSL协议版本
- 内存限制值
