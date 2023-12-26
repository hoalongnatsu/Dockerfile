# Dockerfile for many programming languages

![picture](./images/docker.png)

## Contents
- [React](#dockerfile-for-react)
- [NodeJS](#dockerfile-for-nodejs)
- [Python](#dockerfile-for-python)
- [Golang](#dockerfile-for-golang)
- [Java Spring Boot](#dockerfile-for-java-spring-boot)
- [Java Quarkus](#dockerfile-for-java-quarkus)
- [Rust](#dockerfile-for-rust)
- [PHP Laravel](#dockerfile-for-php-laravel)
- [Contact](#contact)

## Dockerfile for React
Normal:

```
FROM node:20-alpine as build

WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:stable-alpine
COPY --from=build /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

With pnpm:

```
FROM node:20-alpine as build

RUN npm install -g pnpm

WORKDIR /app
COPY package*.json ./
RUN CI=true pnpm install
COPY . .
RUN pnpm build

FROM nginx:stable-alpine
COPY --from=build /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

Nginx Config:

```
# auto detects a good number of processes to run
worker_processes auto;

#Provides the configuration file context in which the directives that affect connection processing are specified.
events {
    # Sets the maximum number of simultaneous connections that can be opened by a worker process.
    worker_connections 8000;
    # Tells the worker to accept multiple connections at a time
    multi_accept on;
}


http {
    # what times to include
    include       /etc/nginx/mime.types;
    # what is the default one
    default_type  application/octet-stream;

    # Sets the path, format, and configuration for a buffered log write
    log_format compression '$remote_addr - $remote_user [$time_local] '
        '"$request" $status $upstream_addr '
        '"$http_referer" "$http_user_agent"';

    server {
        # listen on port 80
        listen 80;
        # save logs here
        access_log /var/log/nginx/access.log compression;

        # where the root here
        root /usr/share/nginx/html;
        # what file to server as index
        index index.html index.htm;

        location / {
            # First attempt to serve request as file, then
            # as directory, then fall back to redirecting to index.html
            try_files $uri $uri/ /index.html;
        }

        # Media: images, icons, video, audio, HTC
        location ~* \.(?:jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|mp4|ogg|ogv|webm|htc)$ {
          expires 1M;
          access_log off;
          add_header Cache-Control "public";
        }

        # Javascript and CSS files
        location ~* \.(?:css|js)$ {
            try_files $uri =404;
            expires 1y;
            access_log off;
            add_header Cache-Control "public";
        }

        # Any route containing a file extension (e.g. /devicesfile.js)
        location ~ ^.+\..+$ {
            try_files $uri =404;
        }
    }
}
```

## Dockerfile for NodeJS
ExpressJS:
```
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci
COPY . .

CMD ["node", "index.js"]
```

NestJS Framework:
```
FROM node:20-alpine as build

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY --chown=node:node . .
RUN npm run build

# Running `npm ci` removes the existing node_modules directory and passing in --only=production ensures that only the production dependencies are installed. This ensures that the node_modules directory is as optimized as possible
RUN npm ci --only=production && npm cache clean --force

USER node

FROM node:20-alpine

WORKDIR /app

COPY --from=build --chown=node:node /app/package*.json ./
COPY --from=build --chown=node:node /app/node_modules ./node_modules
COPY --from=build --chown=node:node /app/dist ./dist

CMD ["node", "dist/main.js"]
```

## Dockerfile for Python
Normal:
```
FROM python:3.9-slim-bullseye

WORKDIR /app

COPY requirements.txt requirements.txt
RUN pip3 install -r requirements.txt

COPY . .

CMD ["python3", "app.py"]
```

With Flask or Diango, you need to run on host `0.0.0.0`.

Flask:
```
FROM python:3.9-slim-bullseye

WORKDIR /app

COPY requirements.txt requirements.txt
RUN pip3 install -r requirements.txt

COPY . .

CMD [ "python3", "-m" , "flask", "run", "--host=0.0.0.0"]
```

Diango:
```
FROM python:3.9-slim-bullseye

WORKDIR /app

COPY requirements.txt requirements.txt
RUN pip3 install -r requirements.txt

COPY . .

CMD ["python3", "manage.py", "runserver", "0.0.0.0:8000"]
```

## Dockerfile for Golang
Normal:

```
FROM golang:1.20-alpine AS build

WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download && go mod verify
COPY . .

# Builds the application as a staticly linked one, to allow it to run on alpine
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o run .

# Moving the binary to the 'final Image' to make it smaller
FROM alpine
WORKDIR /app
COPY --from=build /build/run .
CMD ["/app/run"]
```

With private repo:

```
FROM golang:1.20-alpine AS build

# Install git and openssh
RUN apt update && apt upgrade -y && \
    apt install -y git make openssh-client

WORKDIR /build
COPY go.mod go.sum ./
RUN go mod download && go mod verify
COPY . .

# Builds the application as a staticly linked one, to allow it to run on alpine
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o run .

# Moving the binary to the 'final Image' to make it smaller
FROM alpine
WORKDIR /app
COPY --from=build /build/run .
CMD ["/app/run"]
```

## Dockerfile for Java Spring Boot

```
FROM eclipse-temurin:17-jdk-focal as build
 
WORKDIR /build

COPY .mvn/ ./.mvn
COPY mvnw pom.xml  ./
RUN ./mvnw dependency:go-offline

COPY . .
RUN ./mvnw package -DskipTests

FROM eclipse-temurin:17-jdk-alpine
WORKDIR /app
COPY --from=build /build/target/*.jar run.jar
ENTRYPOINT ["java", "-jar", "/app/run.jar"]
```

## Dockerfile for Java Quarkus

```
FROM maven:3.8.4-openjdk-17 AS build

WORKDIR /build
COPY ./pom.xml ./pom.xml
COPY ./settings.xml /root/.m2/settings.xml
RUN mvn dependency:go-offline -B

COPY src src
ARG QUARKUS_PROFILE
RUN mvn package -Dquarkus.profile=${QUARKUS_PROFILE}

FROM registry.access.redhat.com/ubi8/ubi-minimal:8.4 

ARG JAVA_PACKAGE=java-17-openjdk-headless
ARG RUN_JAVA_VERSION=1.3.8
ENV LANG='en_US.UTF-8' LANGUAGE='en_US:en'

# Install java and the run-java script
RUN microdnf install curl ca-certificates wget ${JAVA_PACKAGE} \
  && microdnf update \
  && microdnf clean all \
  && mkdir /deployments \
  && chown 1001 /deployments \
  && chmod "g+rwX" /deployments \
  && chown 1001:root /deployments \
  && curl https://repo1.maven.org/maven2/io/fabric8/run-java-sh/${RUN_JAVA_VERSION}/run-java-sh-${RUN_JAVA_VERSION}-sh.sh -o /deployments/run-java.sh \
  && chown 1001 /deployments/run-java.sh \
  && chmod 540 /deployments/run-java.sh \
  && echo "securerandom.source=file:/dev/urandom" >> /etc/alternatives/jre/conf/security/java.security
RUN mkdir /javaagent && \
  wget -O /javaagent/elastic-apm-agent-1.30.0.jar https://search.maven.org/remotecontent?filepath=co/elastic/apm/elastic-apm-agent/1.30.0/elastic-apm-agent-1.30.0.jar

# We make four distinct layers so if there are application changes the library layers can be re-used
COPY --from=build /build/target/quarkus-app/lib/ /deployments/lib/
COPY --from=build /build/target/quarkus-app/*.jar /deployments/
COPY --from=build /build/target/quarkus-app/app/ /deployments/app/
COPY --from=build /build/target/quarkus-app/quarkus/ /deployments/quarkus/

USER 1001

ENTRYPOINT [ "/deployments/run-java.sh" ]
```

## Dockerfile for Rust
Normal:

```
FROM rust:1.70.0-slim-bullseye AS build

# View app name in Cargo.toml
ARG APP_NAME=devopsvn

WORKDIR /build

COPY Cargo.lock Cargo.toml ./
RUN mkdir src \
    && echo "// dummy file" > src/lib.rs \
    && cargo build --release

COPY src src
RUN cargo build --locked --release
RUN cp ./target/release/$APP_NAME /bin/server

FROM debian:bullseye-slim AS final
COPY --from=build /bin/server /bin/
ENV ROCKET_ADDRESS=0.0.0.0
CMD ["/bin/server"]
```

With non-privileged user:

```
FROM rust:1.70.0-slim-bullseye AS build

# View app name in Cargo.toml
ARG APP_NAME=devopsvn

WORKDIR /build

COPY Cargo.lock Cargo.toml ./
RUN mkdir src \
    && echo "// dummy file" > src/lib.rs \
    && cargo build --release

COPY src src
RUN cargo build --locked --release
RUN cp ./target/release/$APP_NAME /bin/server

FROM debian:bullseye-slim AS final

RUN adduser \
    --disabled-password \
    --gecos "" \
    --home "/nonexistent" \
    --shell "/sbin/nologin" \
    --no-create-home \
    --uid "10001" \
    appuser
USER appuser

COPY --from=build /bin/server /bin/
ENV ROCKET_ADDRESS=0.0.0.0
CMD ["/bin/server"]
```

## Dockerfile for PHP Laravel
Normal:

```
FROM php:8.2-fpm

ARG user
ARG uid

# Install system dependencies
RUN apt-get update && apt-get install -y \
    git \
    curl \
    libpng-dev \
    libonig-dev \
    libxml2-dev \
    zip \
    unzip \
    supervisor \
    nginx \
    build-essential \
    openssl

RUN docker-php-ext-install gd pdo pdo_mysql sockets

# Get latest Composer
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

# Create system user to run Composer and Artisan Commands
RUN useradd -G www-data,root -u $uid -d /home/$user $user
RUN mkdir -p /home/$user/.composer && \
    chown -R $user:$user /home/$user

WORKDIR /var/www

# If you need to fix ssl
COPY ./openssl.cnf /etc/ssl/openssl.cnf

COPY composer.json composer.lock ./
RUN composer install --no-dev --optimize-autoloader --no-scripts

COPY . .

RUN chown -R $uid:$uid /var/www

# copy supervisor configuration
COPY ./supervisord.conf /etc/supervisord.conf

# run supervisor
CMD ["/usr/bin/supervisord", "-n", "-c", "/etc/supervisord.conf"]
```

supervisord.conf:

```
[supervisord]
user=root
nodaemon=true
logfile=/dev/stdout
logfile_maxbytes=0
pidfile=/var/run/supervisord.pid
loglevel = INFO

[program:php-fpm]
command = /usr/local/sbin/php-fpm
autostart=true
autorestart=true
priority=5
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0

[program:nginx]
command=/usr/sbin/nginx -g "daemon off;"
autostart=true
autorestart=true
priority=10
stdout_events_enabled=true
stderr_events_enabled=true
stdout_logfile=/dev/stdout
stdout_logfile_maxbytes=0
stderr_logfile=/dev/stderr
stderr_logfile_maxbytes=0
```

openssl.cnf:

```
openssl_conf = openssl_init

[openssl_init]
ssl_conf = ssl_sect

[ssl_sect]
system_default = system_default_sect

[system_default_sect]
Options = UnsafeLegacyRenegotiation
```

If you want add more extension, for example MongoDB, create a file php.ini:

```
extension=mongodb.so
```

Update Dockerfile:

```
...
WORKDIR /var/www

# If you need to fix ssl
COPY ./openssl.cnf /etc/ssl/openssl.cnf
# If you need add extension
COPY ./php.ini /usr/local/etc/php/php.ini
...
```

## Contact
Need update? Contact me on [LinkedIn](https://www.linkedin.com/in/hmquan1996/).
