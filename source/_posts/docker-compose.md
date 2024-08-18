---
type: story
cover: /assets/posts/farewell-azeroth/farewell-azeroth-00.jpg
banner: /assets/posts/farewell-azeroth/farewell-azeroth-00.jpg
poster:
  headline: docker compose
  # topic: 标题上方的小字 
  # color: red
  caption: 正确的打开方式！
title: docker compose使用
description: docker compose使用
categories: [it]
tags: [it, docker]
date: 2024-08-17 10:32:32
---

# docker-compose

### **操作compose项目中的单个服务（start stop restart）**

```
docker compose -p composetest restart web

```

### **compose中的服务和本地文件的相互拷贝**

```
容器 => 本地
docker compose cp my-service:~/path/to/myfile ~/local/path/to/copied/file

本地 => 容器 --all表示所有my-service ?
docker compose cp --all ~/local/path/to/source/file my-service:~/path/to/copied/file

```

### **在compose文件中可以设置环境变量**

```
1.env_file用于指定环境文件.env的路径
2.environment是设置环境

cat ./Docker/api/api.env
NODE_ENV=test

cat docker-compose.yml
version: '3'
services:
  api:
    image: 'node:6-alpine'
    env_file:
     - ./Docker/api/api.env
    environment:
     - NODE_ENV=production

如果几种方法都配置了相同的环境变量，则存在一个优先级（从高到低）
1.使用-env直接在命令行指定：docker compose run --env <KEY[=[VAL]]>
2.在compose文件中指定environment
3.在compose文件中指定env_file
4.docker文件中直接设置ENV, ENV MY_NAME="John Doe"

```

### **通过profile控制要启动的服务**

### **基础版**

```
version: "3.9"
services:
  frontend:
    image: frontend
    profiles: ["frontend"]

  phpmyadmin:
    image: phpmyadmin
    depends_on:
      - db
    profiles:
      - debug

  backend:
    image: backend

  db:
    image: mysql

docker compose up                  //启动backend, db
docker compose --profile debug up  //启动backend, db, phpmyadmin

//直接运行配置了profile的服务, 会隐式开启对应的profile,对应的依赖也会被启动
docker compose run phpmyadmin      //启动db, phpmyadmin

```

### **进阶版**

```
当一个服务的依赖项也是配置了profile，这是就需要两个在同一个profile，才能顺利启动

version: "3.9"
services:
  web:
    image: web

  mock-backend:
    image: backend
    profiles: ["dev"]
    depends_on:
      - db

  db:
    image: mysql
    profiles: ["dev"]

  phpmyadmin:
    image: phpmyadmin
    profiles: ["debug"]
    depends_on:
      - db

docker compose up                    //只启动web
docker compose up -d mock-backend    //启动mock-backend,db (因为两个都具备 dev profile)

//启动失败，因为 phpmyadmin指定的profile=debug, 而它的依赖项db指定的profile=dev
docker compose up -d phpmyadmin

//修复方法
db:
  image: mysql
  profiles: ["debug", "dev"]

```

### **在生产环境中发布更新**

```
docker compose build web
docker compose up --no-deps -d web

单独对web服务进行关闭，销毁，重新创建服务
--no-deps 标记用于单独启动web服务，不会把web的依赖服务也重新启动
```