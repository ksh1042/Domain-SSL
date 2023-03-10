# Domain-SSL
- 도메인 및 SSL 인증서 발급 및 적용에 관한 팁

## 1. 도메인 발급
- 내용 추가 필요
---
## 2. 인증서 발급
- 인증서 발급 절차 중 CNAME 필드의 이름과 값을 복사하여, 도메인을 발급처에 해당 레코드를 추가한 뒤 인증을 받는다.
- 인증이 되었다면 인증서를 내보내기 하여 인증서 파일을 받는다. (pem, crt, key 등의 파일)
<br>※ AWS 인증서의 경우 내보내기가 불가능하므로 간단히 EC2에 적용할 방법은 없다.

## 3. 인증서 적용
### 3.1. Apache httpd for Linux
- apahce httpd mod_ssl 라이브러리를 설치한다.
- /etc/httpd/cert/ 경로를 생성 후 앞서 인증받은 인증서를 root 권한으로 복사하여 해당 디렉터리로 복사한다.
- /etc/httpd/conf.d/ 경로에 http 80 으로 접근하는 부분에 대한 가상 호스트 .conf 파일을 작성한다. (아래의 경우 http로 접근할시 https로 자동 변경됨)
```xml
<VirtualHost *:80>
        ServerName abc.com

        <IfModule mod_rewrite.c>
                RewriteEngine On
                RewriteCond %{HTTPS} off
                RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI}
        </IfModule>
</VirtualHost>
```
- /etc/httpd/conf.d/ 경로에 https 443 으로 접근하는 부분에 대한 가상 호스트 .conf 파일을 작성한다.
```xml
<IfModule ssl_module>
        <VirtualHost _default_:443>
                ServerName abc.com
                DocumentRoot /var/www/abc

                ErrorLog /var/log/httpd/error_log
                CustomLog /var/log/httpd/access_log combined

                SSLEngine on
                SSLCertificateFile /etc/httpd/cert/certificate.crt
                SSLCertificateChainFile /etc/httpd/cert/ca_bundle.crt
                SSLCertificateKeyFile /etc/httpd/cert/private.key

                <Directory /var/www/abc>
                        <IfModule rewrite_module>
                                RewriteEngine On
                                RewriteBase /
                                RewriteRule ^index\.html$ - [L]
                                RewriteCond %{REQUEST_FILENAME} !-f
                                RewriteCond %{REQUEST_FILENAME} !-d
                                RewriteRule . /index.html [L]
                        </IfModule>
                </Directory>
        </VirtualHost>
</IfModule>
```
```yaml
ServerName : 도메인
DocumentRoot /var/www/abc :  웹 서비스가 실행되는 root 위치
  
SSLEngine : ssl 사용 여부
SSLCertificateFile : 공개 인증서 파일 위치
SSLCertificateChainFile : CA 체인 인증서 파일 위치 
SSLCertificateKeyFile : 개인 키 파일 위치

<Directory /var/www/ez2archive> : 웹 서비스가 실행되는 root 위치
```
---
## 4. 서브 도메인 설정
- /etc/httpd/conf.d/ 경로에 http 80 으로 접근하는 부분에 대한 가상 호스트 .conf 파일을 작성한다. (아래의 경우 http로 접근할시 https로 자동 변경됨)
```xml
<VirtualHost *:80>
        ServerName api.abc.com

        <IfModule mod_rewrite.c>
                RewriteEngine On
                RewriteCond %{HTTPS} off
                RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI}
        </IfModule>
</VirtualHost>
```
 ※ ServerName 내용이 서브 도메인으로 지정되었는지 주의한다.
- /etc/httpd/conf.d/ 경로에 https 443 으로 접근하는 부분에 대한 프록시 가상 호스트 .conf 파일을 작성한다. (아래의 경우 같은 서버를 공유하는 환경으로 작성됨)
```xml
<IfModule mod_ssl.c>
        <VirtualHost *:443>
                ServerName api.abc.com

                SSLProxyEngine On
                ProxyRequests Off
                ProxyPreserveHost On
                ProxyPass / http://127.0.0.1:54856/ nocanon
                ProxyPassReverse / http://127.0.0.1:54856/

                RequestHeader set X-Forwarded-Proto "https"
                RequestHeader set X-Forwarded-Port "443"

                SSLEngine On
                SSLCertificateFile /etc/httpd/cert/api_certificate.crt
                SSLCertificateChainFile /etc/httpd/cert/api_ca_bundle.crt
                SSLCertificateKeyFile /etc/httpd/cert/api_private.key
        </VirtualHost>
</IfModule>

```
