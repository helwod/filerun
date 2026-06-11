### FileRun
This repository contains a Docker image for FileRun (version 20220519), which is the last available free version of the FileRun product that was released several years ago. As FileRun has transitioned to a commercial model, where only the latest versions require a purchase, this image serves as a stable reference point for users who need access to this earlier version.

### Overview
FileRun is a robust file management system that offers a range of features to enhance productivity and collaboration. This Docker image allows you to quickly set up FileRun in a containerized environment, leveraging Docker’s capabilities to streamline deployment and management. With this image, you can access essential functionalities without the need for ongoing updates that may disrupt your workflow.

### Docker Compose Configuration
Below is the docker-compose.yml configuration for setting up the FileRun environment, which includes the necessary services such as the database, web server, Tika for document processing, and Elasticsearch for enhanced search capabilities.

```yaml
version: "3.8"

volumes:
  db:
    driver: local
  user-files:
    driver: local
  filerun_www:
    driver: local
  filerun_config:
    driver: local
  esearch:
    driver: local
  onlyoffice-data:
  onlyoffice-logs:
  onlyoffice-lib:
  onlyoffice-postgresql:
  onlyoffice-rabbitmq:
  onlyoffice-redis:
  onlyoffice-fonts:

services:
  db:
    image: mariadb:10.5
    container_name: filerun_mariadb
    restart: always
    environment:
      TZ: Asia/Shanghai
      MYSQL_ROOT_PASSWORD: root
      MYSQL_USER: filerun
      MYSQL_PASSWORD: filerun
      MYSQL_DATABASE: filerun
    volumes:
      - db:/var/lib/mysql

  web:
    image: mrizkihidayat66/filerun
    container_name: filerun_web
    restart: always
    environment:
      TZ: Asia/Shanghai
      FR_DB_HOST: db
      FR_DB_PORT: 3306
      FR_DB_NAME: filerun
      FR_DB_USER: filerun
      FR_DB_PASS: filerun
      APACHE_RUN_USER: www-data
      APACHE_RUN_USER_ID: 33
      APACHE_RUN_GROUP: www-data
      APACHE_RUN_GROUP_ID: 33
      ONLYOFFICE_API_URL: http://onlyoffice:80  # 容器内通信用内部主机名
      ONLYOFFICE_JWT_SECRET: 12345  # 🔐 必须与 above 一致
    depends_on:
      - db
    ports:
      - 8082:80
    links:
      - db:db
    volumes:
      - user-files:/user-files
      - filerun_config:/config
      - filerun_www:/var/www
      #- ./nginx/conf.d/upload-limit.conf:/etc/nginx/conf.d/upload-limit.conf:ro  #修改上传的文件大小限制 
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost"]
      interval: 30s
      timeout: 10s
      retries: 3

  tika:
    image: apache/tika
    container_name: filerun_tika
    restart: always

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:6.8.23
    container_name: filerun_search
    restart: always
    environment:
      - cluster.name=docker-cluster
      - bootstrap.memory_lock=true
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65535
        hard: 65535
    mem_limit: 1g
    command: >
      sh -c "mkdir -p /usr/share/elasticsearch/data/nodes &&
             chown -R elasticsearch:elasticsearch /usr/share/elasticsearch/data &&
             /usr/local/bin/docker-entrypoint.sh"
    volumes:
      - esearch:/usr/share/elasticsearch/data
 
  onlyoffice:
    #image: lovechen/seafile-pro-mc:onlyoffice-ce-7.0.0.132
    image: lovechen/seafile-pro-mc:onlyoffice-ce-sp-6.3.1.32
    container_name: onlyoffice
    stdin_open: true
    restart: unless-stopped
    ports:
      - "8881:80"  # ✅ 确保宿主机 8888 映射到容器 80
    environment:
      # --- 基础配置 ---
      # --- HTTPS 配置 (初次调试建议注释掉，调通后再开启)（如果你没有配置证书文件挂载，建议先注释掉以下三行，使用 HTTP 测试连通性） ---
      #- ONLYOFFICE_HTTPS_HSTS_ENABLED=True
      #- SSL_CERTIFICATE_PATH=/var/www/onlyoffice/Data/certs/fullchain.cer
      #- SSL_KEY_PATH=/var/www/onlyoffice/Data/certs/private.key
      #- DB_TYPE=${DB_TYPE:-mariadb}
      #- DB_HOST=${SEAFILE_MYSQL_DB_HOST:-db}
      #- DB_USER=${SEAFILE_MYSQL_DB_USER:-filerun}
      #- DB_PWD=${SEAFILE_MYSQL_DB_PASSWORD:?Variable is not set or empty}
      # ✅ 关键修正 2：启用 JWT 保护（生产环境必须开启，防止未授权访问）
      - JWT_ENABLED=true
      - JWT_SECRET=12345
      #- JWT_HEADER=AuthorizationJwt         # ✅  默认使用的 Header名称
      # Redis 配置（确保主机名与下方 redis 服务名一致）
      - REDIS_SERVER_HOST=onlyoffice-redis
      - REDIS_SERVER_PORT=6379
    volumes:
      - onlyoffice-logs:/var/log/onlyoffice
      - onlyoffice-data:/var/www/onlyoffice/Data
      - onlyoffice-lib:/var/lib/onlyoffice
      - onlyoffice-postgresql:/var/lib/postgresql
      - onlyoffice-redis:/var/lib/redis
      - onlyoffice-rabbitmq:/var/lib/rabbitmq
      - onlyoffice-fonts:/usr/share/fonts/truetype/custom
    depends_on:
      onlyoffice-redis:
        condition: service_healthy

    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80/info/info.json"]
      interval: 30s
      retries: 5
      start_period: 60s
      timeout: 10s

    deploy:
      resources:
        limits:
          cpus: '2.0'   # 文档转换吃 CPU，建议给足
          memory: 4096M # 4G 内存比较稳妥

  onlyoffice-redis:
    container_name: filerun-redis
    image: redis 
    restart: unless-stopped

    expose:
      - '6379'
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
networks:
  default:
    driver: bridge

```

### 修改上传的文件大小限制 /etc/nginx/conf.d/upload-limit.conf

```conf
# 限制所有请求体大小（包括上传文件）
client_max_body_size 1000M;

# 可选：调整客户端超时时间（大文件上传建议延长）
client_body_timeout 300s;
proxy_read_timeout 300s;

```

### Website Setup
<img src="images/6QZhRvNpM1ZnzwkSfK11ERpX.jpg">
<img src="images/RyOxSYmp1cMjdmsJFeGZENrG.jpg">

### Gratitude and Support
We extend our sincere gratitude to FileRun, LDA for their hard work and dedication in developing such a useful platform. If you find FileRun beneficial, we highly recommend supporting them by purchasing a license for the latest version. For pricing details, please visit the [FileRun Pricing Page](https://filerun.com/pricing).
