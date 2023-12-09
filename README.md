# Домашнее задание № 7 по теме: "Управление пакетами. Дистрибьюция софта". К курсу Administrator Linux. Professional

## Задание

- Создать свой RPM-пакет
- Создать свой репозиторий и разместить там ранее собранный RPM-пакет

## Выполнение

- Создали Vagrantfile с описанием двух виртуальных машин __repo__ - машина, на которой производится сборка пакета nginx и она же является репозиторием; __websrv__ - тестовый вэб-сервер, на котором необходимо подключить наш репозиторий и установить собранный вэб-сервер

- Логин на сервер __repo__ `vagrant ssh repo`

```bash
cd /root

wget https://nginx.org/packages/centos/8/SRPMS/nginx-1.24.0-1.el8.ngx.src.rpm
rpm -i nginx-1.*

git clone https://github.com/openssl/openssl.git
```

- Установить зависимости необходимые для сборки

```bash
yum-builddep rpmbuild/SPECS/nginx.spec
```

- Добавить в секцию для rhel 8 зависимость от perl-IPC-Cmd - этот модуль perl необходим для сборки последней версии openssl, openssl-devel >= 1.1.1 исключить, т.к. выполняем статическую компиляцию с библиотекой openssl, заголовочные файлы беруться из -I /root/openssl/.openssl/include

```
%if 0%{?rhel} == 8
%define epoch 1
Epoch: %{epoch}
Requires(pre): shadow-utils
Requires: procps-ng
BuildRequires: perl-IPC-Cmd >= 1.02
%define _debugsource_template %{nil}
%endif
```

- В секцию %build добавить опцию --with-openssl=/root/openssl вместо debug. OpenSSL library statically from the source

```
%build
./configure %{BASE_CONFIGURE_ARGS} \
    --with-cc-opt="%{WITH_CC_OPT}" \
    --with-ld-opt="%{WITH_LD_OPT}" \
    --with-openssl=/root/openssl
make %{?_smp_mflags}
%{__mv} %{bdir}/objs/nginx \
    %{bdir}/objs/nginx-debug
./configure %{BASE_CONFIGURE_ARGS} \
    --with-cc-opt="%{WITH_CC_OPT}" \
    --with-ld-opt="%{WITH_LD_OPT}" \
	--with-openssl=/root/openssl
make %{?_smp_mflags}
```

- Правим версию

```
%define base_version 1.24.0
%define base_release 2%{?dist}.otus
```

- Сборка пакета

```bash
rpmbuild -bb rpmbuild/SPECS/nginx.spec
```

- Установка созданного пакета

```bash
yum localinstall -y rpmbuild/RPMS/x86_64/nginx-1.24.0-2.el8.ngx.x86_64.rpm
```

- Проверяем опции сборки:

```bash
[root@ropository ~]# nginx -V
nginx version: nginx/1.24.0
built by gcc 8.5.0 20210514 (Red Hat 8.5.0-4) (GCC)
built with OpenSSL 3.3.0-dev
TLS SNI support enabled
configure arguments: --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib64/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-O2 -g -pipe -Wall -Werror=format-security -Wp,-D_FORTIFY_SOURCE=2 -Wp,-D_GLIBCXX_ASSERTIONS -fexceptions -fstack-protector-strong -grecord-gcc-switches -specs=/usr/lib/rpm/redhat/redhat-hardened-cc1 -specs=/usr/lib/rpm/redhat/redhat-annobin-cc1 -m64 -mtune=generic -fasynchronous-unwind-tables -fstack-clash-protection -fcf-protection -fPIC' --with-ld-opt='-Wl,-z,relro -Wl,-z,now -pie' --with-openssl=/root/openssl
```

- Запустить сервис

```bash
systemctl enable nginx --now
```

- Проверить статус сервиса

```bash
systemctl status nginx
```

- Создать репозиторий с собранным пакетом и RPM-пакетом для установки репозитория Percona-Server

```bash
mkdir /usr/share/nginx/html/repo
cp rpmbuild/RPMS/x86_64/nginx* /usr/share/nginx/html/repo/
wget https://downloads.percona.com/downloads/percona-distribution-mysql-ps/percona-distribution-mysql-ps-8.0.28/binary/redhat/8/x86_64/percona-orchestrator-3.2.6-2.el8.x86_64.rpm -O /usr/share/nginx/html/repo/percona-orchestrator-3.2.6-2.el8.x86_64.rpm
createrepo /usr/share/nginx/html/repo/
```

- Добавить опцию autoindex on для вывода листинга директории

```bash
sed -i "/location \/ {/a autoindex on;" /etc/nginx/conf.d/default.conf
```

- Проверить синтаксис конфигурационного файла

```bash
[root@ropository ~]# nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

- Перечитать конфиг

```bash
nginx -s reload
```


## Проверка на второй виртуальной машине

- Логин на машину __websrv__ `vagrant ssh websrv`

- Проверить доступность репозитория

```bash
curl http://10.111.177.150/repo/
```

```html
<html>
<head><title>Index of /repo/</title></head>
<body>
<h1>Index of /repo/</h1><hr><pre><a href="../">../</a>
<a href="repodata/">repodata/</a>                                          09-Dec-2023 22:49                   -
<a href="nginx-1.24.0-2.el8.otus.x86_64.rpm">nginx-1.24.0-2.el8.otus.x86_64.rpm</a>                 09-Dec-2023 22:48             5061896
<a href="nginx-debuginfo-1.24.0-2.el8.otus.x86_64.rpm">nginx-debuginfo-1.24.0-2.el8.otus.x86_64.rpm</a>       09-Dec-2023 22:48             2731624
<a href="percona-orchestrator-3.2.6-2.el8.x86_64.rpm">percona-orchestrator-3.2.6-2.el8.x86_64.rpm</a>        16-Feb-2022 15:57             5222976
</pre><hr></body>
</html>
```

- Наш репозиторий добавлен в момент поднятия виртуальной машины websrv (scripts/websrv.sh). Аргумент $1 передается из Vagrantfile и содержит ip-адрес нашего сервера с репозиторием

```bash
cat >> /etc/yum.repos.d/otus.repo << EOF
[otus]
name=otus-linux
baseurl=http://${1}/repo
gpgcheck=0
enabled=1
EOF
```

- Убедиться, что репозиторий подключен

```bash
[root@websrv vagrant]# yum repolist enabled | grep otus
otus                            otus-linux
```

- Просмотреть доступные пакеты в репозитории

```bash
yum list available --disablerepo=* --enablerepo=otus
```

- Установить ранее собранный пакет nginx

```bash
yum install nginx --disablerepo=* --enablerepo=otus
```

- Запустить сервис и проверить

```bash
systemctl enable nginx --now
systemctl status nginx

curl http://10.111.177.160
```

