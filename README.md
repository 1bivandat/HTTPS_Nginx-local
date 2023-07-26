---
# STEP BY STEP
## vi /etc/hosts
```
192.168.1.10 test.com.vn
```
## Step 1: Tạo file cung cấp các thiết lập và yêu cầu cần thiết để tạo ra một Certificate Authority.
- mkdir /etc/nginx/certs; cd /etc/nginx/certs
- vi myAwesomeCA.cnf
```
[ req ]
distinguished_name = req_distinguished_name
x509_extensions = root_ca

[ req_distinguished_name ]
countryName = Country Name (2 letter code)
countryName_min = 2
countryName_max = 2
stateOrProvinceName = State or Province Name (full name)
localityName = Locality Name (eg, city)
0.organizationName = Organization Name (eg, company)
organizationalUnitName = Organizational Unit Name (eg, section)
commonName = Common Name (eg, fully qualified host name)
commonName_max = 64
emailAddress = Email Address
emailAddress_max = 64

[ root_ca ]
basicConstraints = critical, CA:true

[ alt_names ]
IP.1 = 192.168.1.10
DNS.1 = test.com.vn

```
## Step 2: Tạo file định nghĩa các thông tin mở rộng cho chứng chỉ máy chủ.
- vi myAwesomeServer.ext
```
subjectAltName = @alt_names
extendedKeyUsage = serverAuth

[alt_names]
IP.1 = 192.168.1.10
DNS.1 = test.com.vn

```
## Step 3: Tạo một chứng chỉ tự ký và khóa riêng tư cho Certificate Authority
```
openssl req -x509 -newkey rsa:2048 -out myAwesomeCA.cer -outform PEM -keyout myAwesomeCA.pvk -days 10000 -verbose -config myAwesomeCA.cnf -nodes -sha256 -subj "/CN=MyCompany CA"
```
## Step 4: Tạo ra một yêu cầu chứng chỉ (CSR) và khóa riêng tư cho máy chủ với tên chung
```
openssl req -newkey rsa:2048 -keyout myAwesomeServer.pvk -out myAwesomeServer.req -subj /CN=localhost -sha256 -nodes
```
## Step 5: Ký chứng chỉ cho yêu cầu chứng chỉ của máy chủ bằng Certificate Authority (CA)
```
openssl x509 -req -CA myAwesomeCA.cer -CAkey myAwesomeCA.pvk -in myAwesomeServer.req -out myAwesomeServer.cer -days 10000 -extfile myAwesomeServer.ext -sha256 -set_serial 0x1111
```
## Step 6: Sử dụng chứng chỉ và khóa riêng tư của máy chủ, cùng với chứng chỉ CA, để khởi chạy một máy chủ SSL/TLS
```
openssl s_server -accept 15000 -cert myAwesomeServer.cer -key myAwesomeServer.pvk -CAfile myAwesomeCA.cer -WWW
```
## Step 7: Cấu hình Nginx reverse proxy để chạy xác thực https dự án
- vi /etc/nginx/sites-available/default
```
server {
    listen 80;
    server_name test.com.vn;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name test.com.vn;

    ssl_certificate /etc/nginx/certs/myAwesomeServer.cer;
    ssl_certificate_key /etc/nginx/certs/myAwesomeServer.pvk;

    location / {
        proxy_pass http://192.168.1.10:8080;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        proxy_ssl_verify off;
    }
}

```
- nginx -t && systemctl restart nginx 
- Import file /etc/nginx/certs/myAwesomeCA.cer vào trong chrome:
- Setting – privacy and sercurity – sercurity - Manage device certificates
