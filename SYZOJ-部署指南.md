# 前言
目前受官方支持的操作系统为 Ubuntu 16.04 与 Ubuntu 18.04。本文中以 Ubuntu 18.04 为例，如果您使用 Ubuntu 16.04，则需要对一些命令中的系统版本进行替换，或者考虑更新你的操作系统。

SYZOJ 系统分为网站端与评测端两部分，尽管这两个部分可以运行在不同的服务器上（就像由我们维护的 LibreOJ 一样），我们仍然建议对 SYZOJ 架构不够熟悉的人将两个部分运行在同一台服务器上。

目前 SYZOJ 的网站端与评测端均未发布稳定版本，并处于活跃的开发当中，我们建议您按照本教程下载安装 GitHub 上的最新开发版，并保持与 GitHub 的同步。

在此教程中，我们假设用户使用 `root` 账户登录到服务器，并在 `/opt/syzoj` 下安装 SYZOJ。

# 安装系统依赖项
这些软件同时被网站端和评测端依赖，如果您将两部分服务运行在不同的服务器上，请分别在两台服务器上进行这个步骤，否则您只需要在唯一的服务器上进行这个步骤。

SYZOJ 的网站端基于 Node.js 编写。使用以下命令安装 Node.js 运行环境与用于安装 SYZOJ 依赖项的软件包管理器 Yarn：

```bash
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add -
echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list
curl -sL https://deb.nodesource.com/setup_8.x | bash -
apt install -y nodejs yarn
```

从 GitHub 下载 SYZOJ 程序需要 Git，使用以下命令安装 Git：

```bash
apt install -y git
```

# 网站端的部署
## 生成随机密钥
以下的步骤中可能多次需要随机秘钥，可由以下命令生成（建议在多处使用不同的随机密钥）：

```bash
echo $(dd if=/dev/urandom | base64 -w0 | dd bs=1 count=20 2>/dev/null)
```

20 个字符通常可以满足我们的安全需求。如果您需要生成其他长度的随机密钥，请将该命令中 `count=20` 的 20 改为其他正整数。

## 安装系统依赖项
尽管在理论上来说，SYZOJ 网站端可以支持包括易于配置的 SQLite 在内的多种数据库软件，我们仍然只实现了对 MariaDB 的支持。其它的大部分数据库软件未经过我们的测试，我们不保证它们可以正常使用。使用以下命令安装 MariaDB 数据库软件：

```bash
apt install software-properties-common
apt-key adv --recv-keys --keyserver hkp://keyserver.ubuntu.com:80 0xF1656F24C74CD1D8
add-apt-repository 'deb [arch=amd64,arm64,ppc64el] http://mirrors.tuna.tsinghua.edu.cn/mariadb/repo/10.3/ubuntu bionic main'
apt update
apt install -y mariadb-server
```

以上命令通过位于北京的服务器安装 MariaDB，如果您的服务器位于中国大陆以外，请将第三行替换为如下命令，以通过位于香港的服务器安装 MariaDB：

```bash
add-apt-repository 'deb [arch=amd64,arm64,ppc64el] http://mariadb.nethub.com.hk/repo/10.3/ubuntu bionic main'
```

安装 MariaDB 时，您可能被要求回答一些问题，如果您不了解 MariaDB，请全部回答默认值（空值）。

使用以下命令安装用于处理评测队列的 RabbitMQ：

```bash
apt install -y rabbitmq-server
```

使用以下命令安装反向代理服务器软件 Nginx：

```bash
apt install -y nginx
```

## 下载
使用以下命令创建存放 SYZOJ 程序的目录并从 GitHub 下载 SYZOJ 网站端。

```bash
mkdir -p /opt/syzoj
cd /opt/syzoj
git clone https://github.com/syzoj/syzoj
mv syzoj web
```

使用以下命令为 SYZOJ 网站端安装依赖项。取决于您的服务器的网络状况，这可能需要较长的时间。

```bash
cd /opt/syzoj/web
yarn
```

## 配置
使用以下命令从配置文件模板创建用于 SYZOJ 网站端的配置文件。

```bash
mkdir -p /opt/syzoj/config
cp /opt/syzoj/web/config-example.json /opt/syzoj/config/web.json
ln -s ../config/web.json /opt/syzoj/web/config.json
```

使用您熟悉的文本编辑器（以 Vim 为例）编辑 `/opt/syzoj/web/config.json`。如果您不熟悉任何命令行下的文本编辑器，可以考虑使用 `nano`，或者将该文件下载到本地进行编辑，并上传到服务器上。

```bash
vim /opt/syzoj/web/config.json
```

如下是您可能需要修改的一些配置项。其中名称加粗的配置项是您很有可能需要修改的。

* **`title`**：网站的标题。显示在网站每个页面的左上角与标题栏中。
* `hostname`：网站端监听的 IP 地址。如果您按照本教程配置 Nginx 反向代理，请保留默认值 `127.0.0.1`，否则，如果您希望 SYZOJ 网站能够从本机之外访问，请改为 `::`。
* `port`：网站端监听的 TCP 端口。如果默认端口 `5283` 被占用，请将其修改为一个未经占用的端口，并通过修改后的端口连接到 SYZOJ 网站端。
* `db`
	* `database`：数据库名。如果您修改了数据库名，请在下一步创建数据库中同时修改。如果您对数据库不熟悉，请保留默认值 `syzoj`。
	* `username`：连接数据库的用户名。如果您修改了数据库名，请在下一步创建数据库中同时修改。如果您对数据库不熟悉，请保留默认值 `syzoj`。
	* **`password`**：连接数据库的密码。为安全起见，请使用随机密钥填写。
	* `host`：数据库服务器地址。如果您将数据库运行在其他服务器上，请修改为其地址（本教程不讨论这种情况），否则请保留默认值 `127.0.0.1`。
* **`session_secret`**：为安全起见，请使用随机密钥填写。
* **`google_analytics`**：如果您使用 Google Analytics 统计您的 SYZOJ 网站访问数据，请设置为 Google 提供的形如 `UA-XXXXXXXX-X` 的字符串。保留默认值将禁用 Google Analytics。

使用以下命令创建独立的目录用于存放数据和临时文件，这将便于您对网站的维护：

```bash
mv /opt/syzoj/web/uploads /opt/syzoj/data
ln -s ../data /opt/syzoj/web/uploads
mkdir /opt/syzoj/sessions
ln -s ../sessions /opt/syzoj/web/sessions
```

## 创建账户
为安全起见，我们不推荐在生产环境中使用 `root` 账户运行 SYZOJ 网站端。使用以下命令创建一个名为 `syzoj` 的普通账户，并授予它访问 SYZOJ 网站端程序、数据以及配置文件的权限。

```bash
adduser --disabled-password --gecos "" syzoj
chown -R syzoj:syzoj /opt/syzoj/data /opt/syzoj/sessions /opt/syzoj/config/web.json
```

## 创建数据库
执行如下命令启动 MariaDB 客户端：

```bash
mysql
```

在 MariaDB 客户端中执行以下命令创建数据库以及 SYZOJ 网站端连接数据库所使用的用户。

```mysql
CREATE DATABASE `syzoj` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci
GRANT ALL PRIVILEGES ON `syzoj`.* TO "syzoj"@"localhost" IDENTIFIED BY "password";
```

以上命令中的 `password` 为 SYZOJ 网站端连接数据库的密码，请使用与上文配置中相同的值。

进行完此步骤后，按下 <kbd>Ctrl</kbd> + <kbd>D</kbd> 以退出 MariaDB 客户端。

## 使用 systemd
创建 `/etc/systemd/system/syzoj-web.service` 文件，填入如下内容：

```ini
[Unit]
Description=SYZOJ web service
After=network.target mysql.service rc-local.service
Require=mysql.service rc-local.service

[Service]
Type=simple
WorkingDirectory=/opt/syzoj/web
User=syzoj
Group=syzoj
ExecStart=/usr/bin/env NODE_ENV=production /usr/bin/node /opt/syzoj/web/app.js -c /opt/syzoj/config/web.json
Restart=always

[Install]
WantedBy=multi-user.target
```

执行以下命令启动 SYZOJ 网站端，并将其设置为开机自动启动。

```bash
systemctl start syzoj-web
systemctl enable syzoj-web
```

## 使用 Nginx
创建 `/etc/nginx/sites-enabled/syzoj` 文件，填入如下内容：

```nginx
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

server {
    listen [::]:80;
    
    server_name syzoj.example.com;

    location / {
        proxy_http_version 1.1;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Host $host;
        proxy_set_header Connection $connection_upgrade;
        proxy_pass http://127.0.0.1:5283;
    }
}
```

修改上述文件中的 `syzoj.example.com` 为访问您的 SYZOJ 网站所用的域名。如果您希望通过访问服务器 IP 地址或任意指向服务器 IP 地址的域名来访问 SYZOJ，请删除 `/etc/nginx/sites-enabled/default` 文件，并将域名修改为 `_`。

如果您修改了 SYZOJ 网站端监听的端口号，请同时修改上述文件中 `http://127.0.0.1:5283` 的 5283 为与之相同的端口号。

更多关于 Nginx 的配置（如 HTTPS 支持）不在本文的讨论范围内，如有需要请自行查阅相关资料。

# 评测端的部署
未完待续