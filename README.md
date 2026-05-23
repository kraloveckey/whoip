# 🌐 whoip – Who is this IP?

Self-hosted solution to display IP information with Nginx, MaxMind and GeoIP2 databases.

This service returns plain text or JSON responses with IP address, geolocation, ASN, and user agent data – perfect for quick lookups or integrations.

---

## 🔧 Requirements

- A Linux-based host.
- [`MaxMind`](https://www.maxmind.com/en/account/sign-in) account and license key.
- Compiled Nginx with [`nginx-geoip2`](https://github.com/kraloveckey/nginx-geoip2).
- SSL certificate (e.g. via Let's Encrypt).

---

## 📦 GeoIP2 Database Setup

1. Create a MaxMind account and generate a [`license key`](https://www.maxmind.com/en/accounts/YOUR_ACCOUNT_ID_HERE/license-key).

2. Manually download databases:

```shell
$ wget "https://download.maxmind.com/app/geoip_download?license_key=YOUR_LICENSE_KEY&edition_id=GeoLite2-City&suffix=tar.gz"
$ wget "https://download.maxmind.com/app/geoip_download?license_key=YOUR_LICENSE_KEY&edition_id=GeoLite2-ASN&suffix=tar.gz"
$ wget "https://download.maxmind.com/app/geoip_download?license_key=YOUR_LICENSE_KEY&edition_id=GeoLite2-Country&suffix=tar.gz"
```

Or install `geoipupdate` for regular updates:

```shell
$ apt install geoipupdate
```

Configure it via [`/etc/GeoIP.conf`](./GeoIP.conf), and add a cron job:

```shell
$ crontab -e

# Update every Tuesday at 02:00
0 2 * * 2 /usr/bin/geoipupdate
```

---

## ⚙️ Build Nginx with nginx-geoip2

### Option 1: Install nginx from official repository + dynamic module

If you want to use the latest stable nginx without compiling from source, install it from the [official nginx repository](https://nginx.org/en/linux_packages.html) and then build only the dynamic module against it.

#### Debian / Ubuntu

```shell
# Add nginx official repository
$ sudo apt install curl gnupg2 ca-certificates lsb-release ubuntu-keyring
$ curl https://nginx.org/keys/nginx_signing.key | gpg --dearmor \
    | sudo tee /usr/share/keyrings/nginx-archive-keyring.gpg >/dev/null
$ echo "deb [signed-by=/usr/share/keyrings/nginx-archive-keyring.gpg] \
    https://nginx.org/packages/ubuntu $(lsb_release -cs) nginx" \
    | sudo tee /etc/apt/sources.list.d/nginx.list
$ echo -e "Package: *\nPin: origin nginx.org\nPin: release o=nginx\nPin-Priority: 900\n" \
    | sudo tee /etc/apt/preferences.d/99nginx
$ sudo apt update && sudo apt install nginx
```

#### RHEL / CentOS / Rocky / AlmaLinux

```shell
$ sudo yum install yum-utils
$ sudo tee /etc/yum.repos.d/nginx.repo << 'EOF'
[nginx-stable]
name=nginx stable repo
baseurl=https://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
EOF
$ sudo yum install nginx
```

#### Build dynamic module against installed nginx

```shell
$ apt install libmaxminddb0 libmaxminddb-dev mmdb-bin libpcre2-dev libssl-dev build-essential

$ cd /opt
$ git clone https://github.com/kraloveckey/nginx-geoip2.git

# Download the exact nginx version that is installed
$ NGINX_VERSION=$(nginx -v 2>&1 | grep -o '[0-9.]*')
$ wget https://nginx.org/download/nginx-${NGINX_VERSION}.tar.gz
$ tar -xvf nginx-${NGINX_VERSION}.tar.gz
$ cd nginx-${NGINX_VERSION}/

$ ./configure --with-compat --add-dynamic-module=/opt/nginx-geoip2
$ make modules
$ cp objs/ngx_http_geoip2_module.so /usr/lib/nginx/modules/
$ cp objs/ngx_stream_geoip2_module.so /usr/lib/nginx/modules/
```

Add at the top of `/etc/nginx/nginx.conf`:

```nginx
load_module modules/ngx_http_geoip2_module.so;
load_module modules/ngx_stream_geoip2_module.so;
```

---

### Option 2: Compile nginx from source

```shell
$ apt install libmaxminddb0 libmaxminddb-dev mmdb-bin libpcre3 libpcre3-dev libssl-dev

$ cd /opt
$ git clone https://github.com/kraloveckey/nginx-geoip2.git
$ wget https://nginx.org/download/nginx-1.30.2.tar.gz
$ tar -xvf nginx-1.30.2.tar.gz
$ cd nginx-1.30.2/
```

### Static module

```shell
$ ./configure --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-http_v3_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-g -O2 -ffile-prefix-map=/data/builder/debuild/nginx-1.26.2/debian/debuild-base/nginx-1.26.2=. -flto=auto -ffat-lto-objects -flto=auto -ffat-lto-objects -fstack-protector-strong -Wformat -Werror=format-security -Wp,-D_FORTIFY_SOURCE=2 -fPIC' --with-ld-opt='-Wl,-Bsymbolic-functions -flto=auto -ffat-lto-objects -flto=auto -Wl,-z,relro -Wl,-z,now -Wl,--as-needed -pie' --add-module=/opt/nginx-geoip2

$ make
$ make install
```

### Dynamic module

```shell
$ ./configure --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --modules-path=/usr/lib/nginx/modules --conf-path=/etc/nginx/nginx.conf --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --pid-path=/var/run/nginx.pid --lock-path=/var/run/nginx.lock --http-client-body-temp-path=/var/cache/nginx/client_temp --http-proxy-temp-path=/var/cache/nginx/proxy_temp --http-fastcgi-temp-path=/var/cache/nginx/fastcgi_temp --http-uwsgi-temp-path=/var/cache/nginx/uwsgi_temp --http-scgi-temp-path=/var/cache/nginx/scgi_temp --user=nginx --group=nginx --with-compat --with-file-aio --with-threads --with-http_addition_module --with-http_auth_request_module --with-http_dav_module --with-http_flv_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_mp4_module --with-http_random_index_module --with-http_realip_module --with-http_secure_link_module --with-http_slice_module --with-http_ssl_module --with-http_stub_status_module --with-http_sub_module --with-http_v2_module --with-http_v3_module --with-mail --with-mail_ssl_module --with-stream --with-stream_realip_module --with-stream_ssl_module --with-stream_ssl_preread_module --with-cc-opt='-g -O2 -ffile-prefix-map=/data/builder/debuild/nginx-1.26.2/debian/debuild-base/nginx-1.26.2=. -flto=auto -ffat-lto-objects -flto=auto -ffat-lto-objects -fstack-protector-strong -Wformat -Werror=format-security -Wp,-D_FORTIFY_SOURCE=2 -fPIC' --with-ld-opt='-Wl,-Bsymbolic-functions -flto=auto -ffat-lto-objects -flto=auto -Wl,-z,relro -Wl,-z,now -Wl,--as-needed -pie' --with-compat --add-dynamic-module=/opt/nginx-geoip2

$ make modules
$ make install
$ cp objs/ngx_http_geoip2_module.so /usr/lib/nginx/modules/
$ cp objs/ngx_stream_geoip2_module.so /usr/lib/nginx/modules/
```

Add at the top of `/etc/nginx/nginx.conf`:

```nginx
load_module modules/ngx_http_geoip2_module.so;
load_module modules/ngx_stream_geoip2_module.so;
```

---

## 📁 Nginx Configuration

Place the following files in your Nginx folder:

- [`/etc/nginx/ip-template.conf`](./ip-template.conf) – routes and responses.
- [`/etc/nginx/conf.d/geoip2.conf`](./geoip2.conf) – GeoIP2 database bindings.
- [`/etc/nginx/conf.d/ip.conf`](./ip.conf) – Nginx server block.
- [`/etc/nginx/nginx.conf`](./nginx.conf) – the primary configuration file for Nginx.
- [`/etc/nginx/cloudflare`](./cloudflare) – configuration file containing Cloudflare IP addresses for Nginx to use with the realip module and for setting up country checking by IP address.

---

## 🔍 API Endpoints

| Endpoint         | Description                                    |
| ---------------- | ---------------------------------------------- |
| `/`              | Client's IP address                            |
| `/full`          | Full info (IP, country, city, ASN, user agent) |
| `/country`       | Country name                                   |
| `/city`          | City name                                      |
| `/city_location` | City longitude                                 |
| `/asn`           | Autonomous system number                       |
| `/as_desc`       | ASN organization                               |
| `/ua`            | User agent                                     |
| `/json`          | JSON with all available info                   |
| `/build_epoch`   | Build time of GeoIP databases                  |

---

## 🧪 Test

Start Nginx and `curl` one of the endpoints:

```shell
$ curl https://ip.dns.com/full

51.158.xx.xx
FR / France
Paris / 2.34940
AS12876 / Scaleway S.a.s.

curl/8.7.1
```

## Example responses

```shell
$ curl ip.dns.com/json | jq .
{
  "ip": "51.158.xx.xx",
  "country_code": "FR",
  "country_name": "France",
  "city_name": "Paris",
  "city_location": "2.34940",
  "asn": "12876",
  "as_desc": "Scaleway S.a.s.",
  "user_agent": "curl/8.7.1"
}
```

```shell
$ curl ip.dns.com

51.158.xx.xx
```

```shell
$ curl ip.dns.com/ua

Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/137.0.0.0 Safari/537.36
```

```shell
$ curl -s -H "X-Real-IP: 8.8.8.8" "ip.dns.com/country_name"
United States
$ curl -s -H "X-Real-IP: 1.1.8.8" "ip.dns.com/country_name"
China
```