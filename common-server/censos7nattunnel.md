# Centos内网穿透-搭建Ngrok实现

```
 什么是内网穿透？我举个例子，比如你在本地搭建的网站，想让局域网外的人访问，该怎么实现？这里有个很多人都熟知的工具：**花生壳**。最开早我用的是花生壳，一个实现远程连接，一个实现内网穿透。几年后的最近发现了ngrok,就百度实现搭建一把试试。
```

### 1、准备

一台云服务器,一个域名,并且域名泛解析解析到云服务器,此处我用的服务器的操作系统为CentOS7\(amd64\)

### 2、安装环境

安装gcc和git（用于下载ngrok源码）

```
yum install gcc -y

yum install git -y
```

### 3、安装go语言环境

```
yum install -y mercurial git bzr subversion golang golang-pkg-windows-amd64 golang-pkg-windows-386
```

### 4、检查环境安装

```
git --version //(>= 1.7 )

go version
```

### 5、在服务器上搭建Ngrok服务

5.1.下载ngrok源码

```
git clone https://github.com/inconshreveable/ngrok.git
```

5.2.生成证书

```
cd ngrok
```

leo.com这里修改为自己的域名

```
export NGROK_DOMAIN="leo.com"

openssl genrsa -out rootCA.key 2048

openssl req -x509 -new -nodes -key rootCA.key -subj "/CN=$NGROK_DOMAIN" -days 5000 -out rootCA.pem

openssl genrsa -out device.key 2048

openssl req -new -key device.key -subj "/CN=$NGROK_DOMAIN" -out device.csr

openssl x509 -req -in device.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out device.crt -days 5000
```

5.3.将新生成的证书替换，执行下面命令后 “y” 回车 一行一行执行代码！

```
cp rootCA.pem assets/client/tls/ngrokroot.crt

cp device.crt assets/server/tls/snakeoil.crt

cp device.key assets/server/tls/snakeoil.key
```

### 6、编译生成ngrokd（服务端）

```
GOOS=linux GOARCH=amd64 make release-server
```

​ 生成在~/ngrok/bin/目录中

### 7、编译生成ngrok（客户端）

```
# 32位linux客户端: 
GOOS=linux GOARCH=386 make release-client

# 64位linux客户端: 
GOOS=linux GOARCH=amd64 make release-client

#32位windows客户端: 
GOOS=windows GOARCH=386 make release-client

#64位windows客户端: 
GOOS=windows GOARCH=amd64 make release-client

#32位mac平台客户端:
GOOS=darwin GOARCH=386 make release-client

#64位mac平台客户端:
GOOS=darwin GOARCH=amd64 make release-client

#ARM平台linux客户端: 
GOOS=linux GOARCH=arm make release-client
```

​ 生成在~/ngrok/bin/目录中

### 8、开启远程服务

在ngrok目录中执行

```
./bin/ngrokd -domain="leo.com" -httpAddr=":9080" -httpsAddr=":9043"-tunnelAddr=":9083" //端口号自己选择设置
```

### 11、开启客户机服务

一.新建配置文件ngrok.cfg，注意格式,使用yml语法

```
server_addr: "leo.com:9083"
trust_host_root_certs: false
tunnels:
  http:
    subdomain: "www"
    proto:
      http: "80"
  https:
    subdomain: "www"
    proto:
      https: "443"

  ssh:
    remote_port: 9022
    proto:
      tcp: "22"
```

二.执行客户机服务（当前执行客户机系统centos，其它系统自行测试）

```
ngrok -config=ngrok.cfg start http https ssh
```

```
ngrok                                                           (Ctrl+C to quit)
Tunnel Status                 online                                            
Version                       1.7/1.7                                           
Forwarding                    https://www.leo.com:9043 -> 127.0.0.1:443      
Forwarding                    tcp://leo.com:9022 -> 127.0.0.1:22             
Forwarding                    http://www.leo.com:9080 -> 127.0.0.1:80        
Web Interface                 127.0.0.1:4040                                    
# Conn                        4                                              
Avg Conn Time                 49270.69ms
```



