## SSL 证书生成
使用版本：Win64 OpenSSL v3.0.5 Light
</br>
下载网址：https://slproweb.com/products/Win32OpenSSL.html
### root ca
* 生成密钥<br>
`` openssl genrsa -des3 -out ca.key 2048``
* 创建证书请求<br>
  ``openssl req -new -key ca.key -out ca.csr``
* 生成crt<br>
  ``openssl x509 -req -days 365 -in ca.csr -out ca.crt -key ca.key``
### 配置文件修改
* 查询openssl.cnf文件位置<br>
  ``openssl version -d``
* 打开openssl.cnf文件
  * 打开 copy_extensions = copy
  * 打开 req_extensions = v3_req
  * 找到[ v3_req ]，添加 subjectAltName = @alt_names
  * 填写新的标签 [ alt_names ] 和标签字段
> [ alt_names ] </br>
> DNS.1 = *.suoy.net

### 服务端/客户端 密钥

* 生成私钥文件<br>
  `` openssl genpkey -algorithm RSA -out server.key``
* 通过私钥生成 证书请求文件<br>
  `` openssl req -new -nodes -key server.key -out server.csr -days 3650 -config ./openssl.cnf -extensions v3_req``
* 生成生成SAN证书<br>
  ``openssl x509 -req -in ./grpc/server.csr -CA ./grpc/ca.crt -CAkey ./grpc/ca.key -CAcreateserial -out users.pem -days 365 -extfile ./grpc/openssl.cnf -extensions v3_req``

``openssl x509 -req -in ./grpc/server.csr -CA ./grpc/ca.crt -CAkey ./grpc/ca.key -CAcreateserial -out users.pem -days 365 -extfile ./grpc/openssl.cnf -extensions v3_req``
