# Dockerfile for many programming languages

![picture](./images/docker.png)

## Contents
- [React](#dockerfile-for-react)
- [NodeJS](#dockerfile-for-nodejs)
- [Python](#dockerfile-for-python)
- [Golang](#dockerfile-for-golang)
- [Java Spring Boot](#dockerfile-for-java-spring-boot)
- [Java Quarkus](#dockerfile-for-java-quarkus)
- [ASP.NET Core](#dockerfile-for-aspnet-core)
- [Ruby](#dockerfile-for-ruby-on-rails)
- [Rust](#dockerfile-for-rust)
- [PHP Laravel](#dockerfile-for-php-laravel)
- [Dart](#dockerfile-for-dart)
- [R Studio](#dockerfile-for-r-studio) 
- [Contact](#contact)

## Dockerfile for React
Normal:

```Dockerfile
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

```Dockerfile
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
```Dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci
COPY . .

CMD ["node", "index.js"]
```

Install node-gyp on Node Alpine version:
```Dockerfile
FROM node:20-alpine

RUN apk add --no-cache \
    make \
    gcc \
    g++ \
    python3 \
    pkgconfig \
    pixman-dev \
    cairo-dev \
    pango-dev \
    libjpeg-turbo-dev \
    giflib-dev

WORKDIR /app

COPY package*.json ./
RUN npm install
COPY . .

CMD ["node", "index.js"]
```

If you try to use the sharp package with NodeJS but encounter errors:
+ `Could not load the "sharp" module using the linuxmusl-x64 runtime`
+ `sharp: Installation error: Invalid Version: 1.2.4_git20230717`

Fix by changing `FROM node:20-alpine` to `FROM node:20-buster-slim`:
```Dockerfile
FROM node:20-buster-slim

WORKDIR /app

COPY package*.json ./
RUN npm ci
COPY . .

CMD ["node", "index.js"]
```

NestJS Framework:
```Dockerfile
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
```Dockerfile
FROM python:3.9-slim-bullseye

WORKDIR /app

COPY requirements.txt requirements.txt
RUN pip3 install -r requirements.txt

COPY . .

CMD ["python3", "app.py"]
```

With Flask or Django, you need to run on host `0.0.0.0`.

Flask:
```Dockerfile
FROM python:3.9-slim-bullseye

WORKDIR /app

COPY requirements.txt requirements.txt
RUN pip3 install -r requirements.txt

COPY . .

CMD [ "python3", "-m" , "flask", "run", "--host=0.0.0.0"]
```

Django:
```Dockerfile
FROM python:3.9-slim-bullseye

WORKDIR /app

COPY requirements.txt requirements.txt
RUN pip3 install -r requirements.txt

COPY . .

CMD ["python3", "manage.py", "runserver", "0.0.0.0:8000"]
```

With Poetry, python package management like npm in Node:
+ `pyproject.toml` similar to `package.json` in Node
+ `poetry.lock` similar to `package-lock.json` in Node

```Dockerfile
FROM python:3.9-slim-bullseye as builder
RUN pip install poetry

WORKDIR /app
COPY poetry.lock pyproject.toml ./
RUN poetry install

FROM python:3.9-slim-bullseye as base
WORKDIR /app

COPY --from=builder /app /app

ENV PATH="/app/.venv/bin:$PATH"
CMD ["python", "-m", "app.py"]
```


## Dockerfile for Golang
Normal:

```Dockerfile
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

```Dockerfile
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

```Dockerfile
FROM eclipse-temurin:17-jdk-focal as build
 
WORKDIR /build

COPY .mvn/ ./.mvn
COPY mvnw pom.xml  ./
RUN sed -i 's/\r$//' mvnw
RUN ./mvnw dependency:go-offline

COPY . .
RUN sed -i 's/\r$//' mvnw
RUN ./mvnw package -DskipTests

FROM eclipse-temurin:17-jdk-alpine
WORKDIR /app
COPY --from=build /build/target/*.jar run.jar
ENTRYPOINT ["java", "-jar", "/app/run.jar"]
```

## Dockerfile for Java Quarkus

```Dockerfile
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

# We make four distinct layers so if there are application changes the library layers can be re-used
COPY --from=build /build/target/quarkus-app/lib/ /deployments/lib/
COPY --from=build /build/target/quarkus-app/*.jar /deployments/
COPY --from=build /build/target/quarkus-app/app/ /deployments/app/
COPY --from=build /build/target/quarkus-app/quarkus/ /deployments/quarkus/

USER 1001

ENTRYPOINT [ "/deployments/run-java.sh" ]
```

## Dockerfile for ASP.NET Core
Normal:
```Dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build

WORKDIR /build

# copy csproj and restore as distinct layers
COPY *.csproj .
RUN dotnet restore

# copy and publish app and libraries
COPY . .
RUN dotnet publish --no-restore -o app

FROM mcr.microsoft.com/dotnet/aspnet:8.0

WORKDIR /app

COPY --from=build /build/app .

ENTRYPOINT ["./aspnetapp"]
```

Alpine version:
```Dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0-alpine AS build

WORKDIR /build

# copy csproj and restore as distinct layers
COPY *.csproj .
RUN dotnet restore

# copy and publish app and libraries
COPY . .
RUN dotnet publish --no-restore -o app

FROM mcr.microsoft.com/dotnet/aspnet:8.0-alpine

WORKDIR /app

COPY --from=build /build/app .

ENTRYPOINT ["./aspnetapp"]
```
## Dockerfile for Ruby on Rails
Without assets:
```Dockerfile
FROM ruby:3.2-slim-bullseye

# Install system dependencies required both at runtime and build time
RUN apt-get update && apt-get install -y \
    build-essential \
    # example system dependencies that need for "gem install pg"
    libpq-dev

COPY Gemfile Gemfile.lock ./

# Install (excluding development/test dependencies)
RUN gem install bundler && \
  bundle config set without "development test" && \
  bundle install

COPY . .

CMD ["rails", "server", "-b", "0.0.0.0"]
```

With assets:
```Dockerfile
FROM ruby:3.2-slim-bullseye

# Install system dependencies required both at runtime and build time
RUN apt-get update && apt-get install -y \
    build-essential \
    # example system dependencies that need for "gem install pg"
    libpq-dev \
    nodejs \
    yarn

COPY Gemfile Gemfile.lock ./

# Install (excluding development/test dependencies)
RUN gem install bundler && \
  bundle config set without "development test" && \
  bundle install

COPY package.json yarn.lock ./
RUN yarn install

COPY . .

# Install assets
RUN RAILS_ENV=production SECRET_KEY_BASE=assets bundle exec rails assets:precompile

CMD ["rails", "server", "-b", "0.0.0.0"]
```

**Note** - On MacOS M Chip, maybe you need to add a flag `--platform=linux/amd64` when build:

```
docker build . -t rubyonrails-app --platform=linux/amd64
```

## Dockerfile for Rust
Normal:

```Dockerfile
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

```Dockerfile
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

```Dockerfile
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

## Dockerfile for Dart
```Dockerfile
FROM dart AS build

WORKDIR /build

COPY pubspec.* /build
RUN dart pub get --no-precompile

COPY . .
RUN dart compile exe app.dart -o run

FROM debian:bullseye-slim

WORKDIR /build

COPY --from=build /build/run /app/run
CMD ["/app/run"]
```

## Dockerfile for R Studio
SQL Server driver:
```Dockerfile
FROM rocker/rstudio

RUN apt-get update && apt-get install -y \
    curl \
    apt-transport-https \
    tdsodbc \
    libsqliteodbc \
    gnupg \
    unixodbc \
    unixodbc-dev \
    ## clean up
    && apt-get clean \ 
    && rm -rf /var/lib/apt/lists/ \ 
    && rm -rf /tmp/downloaded_packages/ /tmp/*.rds

RUN curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add - \
 && curl https://packages.microsoft.com/config/debian/9/prod.list > /etc/apt/sources.list.d/mssql-release.list \
 && apt-get update \
 && ACCEPT_EULA=Y apt-get install --yes --no-install-recommends msodbcsql17 msodbcsql18 mssql-tools18 \
 && install2.r odbc \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/* \
 && rm -rf /tmp/*

RUN Rscript -e 'install.packages(c("DBI","odbc"))'
```

MYSQL driver:
```Dockerfile
FROM rocker/rstudio

RUN apt-get update && apt-get install -y \
    curl \
    apt-transport-https \
    tdsodbc \
    libsqliteodbc \
    gnupg \
    unixodbc \
    unixodbc-dev \
    ## clean up
    && apt-get clean \ 
    && rm -rf /var/lib/apt/lists/ \ 
    && rm -rf /tmp/downloaded_packages/ /tmp/*.rds

RUN wget https://downloads.mysql.com/archives/get/p/10/file/mysql-connector-odbc-8.0.19-linux-debian9-x86-64bit.tar.gz \
 && tar xvf mysql-connector-odbc-8.0.19-linux-debian9-x86-64bit.tar.gz \
 && cp mysql-connector-odbc-8.0.19-linux-debian9-x86-64bit/bin/* /usr/local/bin \
 && cp mysql-connector-odbc-8.0.19-linux-debian9-x86-64bit/lib/* /usr/local/lib \
 && sudo apt-get update \
 && apt-get install --yes libodbc1 odbcinst1debian2 \
 && chmod 777 /usr/local/lib/libmy*

RUN myodbc-installer -a -d -n "MySQL ODBC 8.0 Driver" -t "Driver=/usr/local/lib/libmyodbc8w.so" \
 && myodbc-installer -a -d -n "MySQL ODBC 8.0" -t "Driver=/usr/local/lib/libmyodbc8a.so"

RUN Rscript -e 'install.packages(c("DBI","odbc"))'
```

## Contact
Need update? Contact me on [LinkedIn](https://www.linkedin.com/in/hmquan1996/).
