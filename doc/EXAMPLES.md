# 설정 예제

Go back to the **[⬆ 목차](https://github.com/dhtmdgkr123/nginx-admins-handbook#table-of-contents)** or **[⬆ What's next?](https://github.com/dhtmdgkr123/nginx-admins-handbook#whats-next)** section.

- **[≡ 설정 예제](#examples)**
  * [역프록시](#역프록시)
    * [설치](#설치)
    * [설정](#설정)
    * [설정 가져오기](#설정-가져오기)
    * [IP주소 지정](#ip주소-지정)
    * [도메인 이름 설정](#도메인-이름-설정)
    * [개인키와 인증서 생성](#개인키와-인증서-생성)
    * [모듈 목록 업데이트](#모듈-목록-업데이트)
    * [에러페이지 생성](#에러페이지-생성)
    * [새 도메인 추가](#새-도메인-추가)
    * [설정 테스트](#설정-테스트)

  > 설정과 파일들을 백업하는걸 잊지 마세요.

이 부분은 아직 작업중입니다.

## 설치

[소스에서 설치](HELPERS.md#소스에서-설치)

## 설정

다음 매개변수와 함께 Google Cloud 인스턴스를 사용:

| <b>ITEM</b> | <b>VALUE</b> | <b>COMMENT</b> |
| :---         | :---         | :---         |
| VM | Google Cloud Platform | |
| vCPU | 2x | |
| Memory | 4096MB | |
| HTTP | Varnish on port 80 | |
| HTTPS | NGINX on port 443 | |

## 역프록시

이 장에서는 내 프록시 서버의 기본 설정에 대해 설명합니다. ([blkcipher.info](https://blkcipher.info) 도메인).

  > 설정은 [소스에서 설치](HELPERS.md#소스에서-설치)장을 기반으로 합니다. [소스에서 설치](HELPERS.md#소스에서-설치)를 따라하셨으면 아래에 나오는 설정을 그대로 쓰실 수 있습니다. (약간의 조정이 필요할 수 있습니다)

#### 설정 가져오기

매우 간단합니다 - 저장소 복제, 현재 설정 백업 그리고 전체 동기화:

```bash
git clone https://github.com/dhtmdgkr123/nginx-admins-handbook

tar czvfp ~/nginx.etc.tgz /etc/nginx && mv /etc/nginx /etc/nginx.old

rsync -avur lib/nginx/ /etc/nginx/
```

  > NGINX를 컴파일 하셨다면 모듈들을 업데이트/새로고침 해야합니다. 컴파일된 모듈들은 이곳에 `/usr/local/src/nginx-${ngx_version}/master/objs` 저장되고 `--modules-path` 값에 따라 설치됩니다.

#### IP주소 지정

###### 파일과 폴더 이름에서 '192.168.252.2' 찾기 및 바꾸기

```bash
cd /etc/nginx
find . -depth -not -path '*/\.git*' -name '*192.168.252.2*' -execdir bash -c 'mv -v "$1" "${1//192.168.252.2/xxx.xxx.xxx.xxx}"' _ {} \;
```

###### 설정파일에서 '192.168.252.2' 찾기 및 바꾸기

```bash
cd /etc/nginx
find . -not -path '*/\.git*' -type f -print0 | xargs -0 sed -i 's/192.168.252.2/xxx.xxx.xxx.xxx/g'
```

#### 도메인 이름 설정

###### 파일과 폴더 이름에서 'blkcipher.info' 찾기 및 바꾸기

```bash
cd /etc/nginx
find . -not -path '*/\.git*' -depth -name '*blkcipher.info*' -execdir bash -c 'mv -v "$1" "${1//blkcipher.info/example.com}"' _ {} \;
```

###### 설정파일에서 'blkcipher.info' 찾기 및 바꾸기

```bash
cd /etc/nginx
find . -not -path '*/\.git*' -type f -print0 | xargs -0 sed -i 's/blkcipher_info/example_com/g'
find . -not -path '*/\.git*' -type f -print0 | xargs -0 sed -i 's/blkcipher.info/example.com/g'
```

#### 개인키와 인증서 생성

###### 로컬호스트용

```bash
cd /etc/nginx/master/_server/localhost/certs

# 개인키와 자체 서명 인증서:
( _fd="localhost.key" ; _fd_crt="nginx_localhost_bundle.crt" ; \
openssl req -x509 -newkey rsa:2048 -keyout ${_fd} -out ${_fd_crt} -days 365 -nodes \
-subj "/C=X0/ST=localhost/L=localhost/O=localhost/OU=X00/CN=localhost" )
```

###### `default_server` 기본 서버용

```bash
cd /etc/nginx/master/_server/defaults/certs

# 개인키와 자체 서명 인증서:
( _fd="defaults.key" ; _fd_crt="nginx_defaults_bundle.crt" ; \
openssl req -x509 -newkey rsa:2048 -keyout ${_fd} -out ${_fd_crt} -days 365 -nodes \
-subj "/C=X1/ST=default/L=default/O=default/OU=X11/CN=default_server" )
```

###### 도메인용 (예. Let's Encrypt)

```bash
cd /etc/nginx/master/_server/example.com/certs

# 다중 도메인:
certbot certonly -d example.com -d www.example.com --rsa-key-size 2048

# 와일드카드 서브 도메인:
certbot certonly --manual --preferred-challenges=dns -d example.com -d *.example.com --rsa-key-size 2048

# 개인키와 인증서체인 복사:
cp /etc/letsencrypt/live/example.com/fullchain.pem nginx_example.com_bundle.crt
cp /etc/letsencrypt/live/example.com/privkey.pem example.com.key
```

#### 모듈 목록 업데이트

모듈 목록을 업데이트하고 설정에 `modules.conf` 파일을 포함합니다:

```bash
_mod_dir="/etc/nginx/modules"

:>"${_mod_dir}.conf"

for _module in $(ls "${_mod_dir}/") ; do echo -en "load_module\t\t${_mod_dir}/$_module;\n" >> "${_mod_dir}.conf" ; done
```

#### 에러페이지 생성

  > In the example (`lib/nginx`) error pages are included from `lib/nginx/master/_static/errors.conf` file.

- default location: `/etc/nginx/html`:
  ```
  50x.html  index.html
  ```
- custom location: `/usr/share/www`:
  ```bash
  cd /etc/nginx/snippets/http-error-pages

  ./httpgen

  # sites/ 디렉터리를 /etc/nginx/html 디렉터리와 동기화 할 수 있습니다:
  #   rsync -var sites/ /etc/nginx/html/
  rsync -var sites/ /usr/share/www/
  ```

#### 새 도메인 추가

###### `nginx.conf` 업데이트

```nginx
# At the end of the file (in 'IPS/DOMAINS' section):
include /etc/nginx/master/_server/domain.com/servers.conf;
include /etc/nginx/master/_server/domain.com/backends.conf;
```

###### 도메인 디렉터리 초기 설정

```bash
cd /etc/nginx/cd master/_server
cp -R example.com domain.com

cd domain.com
find . -not -path '*/\.git*' -depth -name '*example.com*' -execdir bash -c 'mv -v "$1" "${1//example.com/domain.com}"' _ {} \;
find . -not -path '*/\.git*' -type f -print0 | xargs -0 sed -i 's/example_com/domain_com/g'
find . -not -path '*/\.git*' -type f -print0 | xargs -0 sed -i 's/example.com/domain.com/g'
```

#### 로그 디렉터리 생성

```bash
mkdir -p /var/log/nginx/localhost
mkdir -p /var/log/nginx/defaults
mkdir -p /var/log/nginx/others
mkdir -p /var/log/nginx/domains/blkcipher.info

chown -R nginx:nginx /var/log/nginx
```

#### 로그순환 설정

```bash
cp /etc/nginx/snippets/logrotate.d/nginx /etc/logrotate.d/
```

#### 설정 테스트

```bash
nginx -t -c /etc/nginx/nginx.conf
```
