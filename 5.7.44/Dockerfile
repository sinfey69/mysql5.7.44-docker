FROM alpine:3.20.3 AS builder

WORKDIR /usr/src

# 拷贝 MySQL 源码
COPY mysql-5.7.44.tar.gz ./
RUN tar zxf mysql-5.7.44.tar.gz 

# 设置代理环境变量
ENV HTTP_PROXY=http://192.168.10.231:7777
ENV HTTPS_PROXY=http://192.168.10.231:7777
ENV NO_PROXY=localhost,127.0.0.1

# 安装构建依赖
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories; \
    apk update; \
    apk add --no-cache \
    curl \
    gpg \
    build-base \
    cmake \
    ncurses-dev \
    openssl-dev \
    bison \
    libaio-dev \
    libtirpc-dev \
    rpcgen

WORKDIR /usr/src/mysql-5.7.44
# 将修改了boost下载地址的文件覆盖到源码包里，提高boost下载速度
COPY boost.cmake cmake/

# 编译 MySQL
RUN cmake . -DDOWNLOAD_BOOST=1 -DWITH_BOOST=/usr/local/boost && \
    make && \
    make install

FROM alpine:3.20.3

COPY --from=builder /usr/local/mysql/bin/* /usr/local/bin/
# 拷贝docker自运行脚本
COPY docker-entrypoint.sh /usr/local/bin/
COPY my-default.cnf /etc/mysql/my.cnf

# 设置代理环境变量
ENV HTTP_PROXY=http://192.168.10.231:7777
ENV HTTPS_PROXY=http://192.168.10.231:7777
ENV NO_PROXY=localhost,127.0.0.1
# 安装构建依赖
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories; \
    apk update; \
    apk add --no-cache \
    curl \
    bash \
    tzdata \
    openssl \
    coreutils \
    libaio \
    libstdc++

#设置时区
RUN cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime && \
    echo "Asia/Shanghai" > /etc/timezone

# add gosu for easy step-down from root
# https://github.com/tianon/gosu/releases
ENV GOSU_VERSION=1.16
RUN set -eux; \
    # TODO find a better userspace architecture detection method than querying the kernel
    arch="$(uname -m)"; \
    case "$arch" in \
    aarch64) gosuArch='arm64' ;; \
    x86_64) gosuArch='amd64' ;; \
    *) echo >&2 "error: unsupported architecture: '$arch'"; exit 1 ;; \
    esac; \
    curl -fL -o /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$gosuArch"; \
    chmod +x /usr/local/bin/gosu; \
    gosu --version; \
    gosu nobody true

# 创建用户组和用户
RUN set -eux; \
    addgroup -S mysql && adduser -S mysql -G mysql

RUN set -eux; \
    # https://github.com/docker-library/mysql/pull/680#issuecomment-636121520
    # grep -F 'socket=/var/lib/mysql/mysql.sock' /etc/mysql/my.cnf; \
    sed -i 's!^socket=.*!socket=/var/run/mysqld/mysqld.sock!' /etc/mysql/my.cnf; \
    sed -i '$a\log-error=/var/log/mysql/error.log' /etc/mysql/my.cnf; \
    # grep -F 'socket=/var/run/mysqld/mysqld.sock' /etc/mysql/my.cnf; \
    { echo '[client]'; echo 'socket=/var/run/mysqld/mysqld.sock'; } >> /etc/mysql/my.cnf; \
    \
    # make sure users dumping files in "/etc/mysql/conf.d" still works
    # ! grep -F '!includedir' /etc/mysql/my.cnf; \
    { echo; echo '!includedir /etc/mysql/conf.d/'; } >> /etc/mysql/my.cnf; \
    mkdir -p /etc/mysql/conf.d; \
    # 5.7 Debian-based images also included "/etc/mysql/mysql.conf.d" so let's include it too
    { echo '!includedir /etc/mysql/mysql.conf.d/'; } >> /etc/mysql/my.cnf; \
    mkdir -p /etc/mysql/mysql.conf.d; \
    \
    # comment out a few problematic configuration values
    find /etc/mysql/my.cnf /etc/mysql/ -name '*.cnf' -print0 \
    | xargs -0 grep -lZE '^(bind-address|log)' \
    | xargs -rt -0 sed -Ei 's/^(bind-address|log)/#&/'; \
    \
    # ensure these directories exist and have useful permissions
    # the rpm package has different opinions on the mode of `/var/run/mysqld`, so this needs to be after install
    mkdir -p /var/lib/mysql /var/run/mysqld /var/log/mysql; \
    chown -R mysql:mysql /var/lib/mysql /var/run/mysqld /usr/local/bin/docker-entrypoint.sh /etc/mysql /var/log/mysql; \
    # ensure that /var/run/mysqld (used for socket and lock files) is writable regardless of the UID our mysqld instance ends up having at runtime
    chmod 1777 /var/lib/mysql /var/run/mysqld; \
    \
    mkdir /docker-entrypoint-initdb.d; \
    \
    mysqld --version; \
    mysql --version;

# 清理构建依赖
# RUN apk del build-base cmake ncurses-dev openssl-dev bison git libaio-dev

VOLUME /var/lib/mysql

ENTRYPOINT ["docker-entrypoint.sh"]

# 暴露端口
EXPOSE 3306 33060

# 启动 MySQL
CMD ["mysqld"]