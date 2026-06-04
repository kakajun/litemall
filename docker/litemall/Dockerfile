# 多阶段构建：第一阶段使用 Maven 编译项目
FROM maven:3.8.6-openjdk-8-slim AS builder

# 安装 Node.js 18（用于编译前端）
RUN apt-get update && apt-get install -y curl \
    && curl -fsSL https://deb.nodesource.com/setup_18.x | bash - \
    && apt-get install -y nodejs \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build

# 先复制 pom.xml 和各模块 pom.xml，利用 Docker 缓存依赖层
COPY pom.xml .
COPY litemall-core/pom.xml litemall-core/pom.xml
COPY litemall-db/pom.xml litemall-db/pom.xml
COPY litemall-wx-api/pom.xml litemall-wx-api/pom.xml
COPY litemall-admin-api/pom.xml litemall-admin-api/pom.xml
COPY litemall-all/pom.xml litemall-all/pom.xml
COPY litemall-all-war/pom.xml litemall-all-war/pom.xml

# 下载依赖（利用缓存）
RUN mvn dependency:go-offline -B -q || true

# 复制完整源码
COPY litemall-core/src litemall-core/src
COPY litemall-db/src litemall-db/src
COPY litemall-db/sql litemall-db/sql
COPY litemall-db/mybatis-generator litemall-db/mybatis-generator
COPY litemall-wx-api/src litemall-wx-api/src
COPY litemall-admin-api/src litemall-admin-api/src
COPY litemall-all/src litemall-all/src

# 复制前端代码并编译
COPY litemall-admin/package.json litemall-admin/package.json
COPY litemall-admin/public litemall-admin/public
COPY litemall-admin/src litemall-admin/src
COPY litemall-admin/build litemall-admin/build
COPY litemall-admin/.env.deployment litemall-admin/.env.deployment
COPY litemall-admin/babel.config.js litemall-admin/babel.config.js
COPY litemall-admin/jsconfig.json litemall-admin/jsconfig.json
COPY litemall-admin/postcss.config.js litemall-admin/postcss.config.js
COPY litemall-admin/vue.config.js litemall-admin/vue.config.js

RUN cd litemall-admin && npm install --registry=https://registry.npmmirror.com && npm run build:dep

# 编译后端（-DskipTests 跳过测试）
RUN mvn clean package -DskipTests -B -q

# 第二阶段：运行时镜像
FROM eclipse-temurin:8-jre

WORKDIR /app

# 从构建阶段复制 jar 包和外部配置文件
COPY --from=builder /build/litemall-all/target/litemall-all-*-exec.jar litemall.jar
COPY docker/litemall/application.yml application.yml

# 创建存储目录
RUN mkdir -p /storage /logs /backup

EXPOSE 8080

ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-Dfile.encoding=UTF-8","-jar","litemall.jar"]
