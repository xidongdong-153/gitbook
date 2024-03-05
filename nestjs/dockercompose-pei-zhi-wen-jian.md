---
description: 数据库 Meilisearch Redis等配置(Docker版本)
---

# 🛠️ docker-compose配置文件

快速部署一套环境

<details>

<summary>docker-compose</summary>

```yaml
version: '3.8'
services:
  mysql:
    image: mysql:latest
    environment:
      MYSQL_ROOT_PASSWORD: 123456789  # 密码
      MYSQL_DATABASE: ink-api  # 创建的数据库
      MYSQL_USER: xidongdong  # MySQL 用户名
      MYSQL_PASSWORD: 123456789  # 用户密码
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql
    restart: always

  meilisearch:
    image: getmeili/meilisearch:latest
    environment:
      MEILI_MASTER_KEY: ink-api-153  # MeiliSearch秘钥
    ports:
      - "7700:7700"
    volumes:
      - meilisearch_data:/data.ms
    restart: always

volumes:
  mysql_data:
    driver: local
  meilisearch_data:
    driver: local
```

</details>
