## openrestry kafka 高并发，高可用
----
### 安装配置
#### 1)准备openresty依赖:
```
apt-get install libreadline-dev libncurses5-dev libpcre3-dev libssl-dev perl make build-essential  
# 或者  
yum install readline-devel pcre-devel openssl-devel gcc 
```
#### 2)安装编译openresty:
```
#1:安装openresty:  
cd /opt/nginx/ # 安装文件所在目录  
wget https://openresty.org/download/openresty-1.9.7.4.tar.gz  
tar -xzf openresty-1.9.7.4.tar.gz /opt/nginx/  
   
#配置:  
# 指定目录为/opt/openresty,默认在/usr/local。  
./configure --prefix=/opt/openresty \  
            --with-luajit \  
            --without-http_redis2_module \  
            --with-http_iconv_module  
make  
make install  
```
#### 3)安装lua-resty-kafka
```
#下载lua-resty-kafka:  
wget https://github.com/doujiang24/lua-resty-kafka/archive/master.zip  
unzip lua-resty-kafka-master.zip -d /opt/nginx/  
    
#拷贝lua-resty-kafka到openresty  
mkdir /opt/openresty/lualib/kafka  
cp -rf /opt/nginx/lua-resty-kafka-master/lib/resty /opt/openresty/lualib/kafka/  
```
#### 4):安装单机kafka
```
cd /opt/nginx/  
wget http://apache.fayea.com/kafka/0.9.0.1/kafka_2.10-0.9.0.1.tgz  
tar xvf kafka_2.10-0.9.0.1.tgz  
   
# 开启单机zookeeper  
nohup sh bin/zookeeper-server-start.sh config/zookeeper.properties > ./zk.log 2>&1 &  
# 绑定broker ip,必须绑定  
#在config/servier.properties下修改host.name  
#host.name={your_server_ip}  
# 启动kafka服务  
nohup sh bin/kafka-server-start.sh config/server.properties > ./server.log 2>&1 &  
# 创建测试topic  
sh bin/kafka-topics.sh --zookeeper localhost:2181 --create --topic test1 --partitions 1 --replication-factor 1  
```
#### 五:配置运行
开发编辑/opt/openresty/nginx/conf/nginx.conf 实现kafka记录nginx日志功能，源码如下:
```
worker_processes  12;  
   
events {  
    use epoll;  
    worker_connections  65535;  
}  
   
http {  
    include       mime.types;  
    default_type  application/octet-stream;  
    sendfile        on;  
    keepalive_timeout  0;  
    gzip on;  
    gzip_min_length  1k;  
    gzip_buffers     4 8k;  
    gzip_http_version 1.1;  
    gzip_types       text/plain application/x-javascript text/css application/xml application/X-JSON;  
    charset UTF-8;  
    # 配置后端代理服务  
    upstream rc{  
        server 10.10.*.15:8080 weight=5 max_fails=3;  
        server 10.10.*.16:8080 weight=5 max_fails=3;  
        server 10.16.*.54:8080 weight=5 max_fails=3;  
        server 10.16.*.55:8080 weight=5 max_fails=3;  
        server 10.10.*.113:8080 weight=5 max_fails=3;  
        server 10.10.*.137:8080 weight=6 max_fails=3;  
        server 10.10.*.138:8080 weight=6 max_fails=3;  
        server 10.10.*.33:8080 weight=4 max_fails=3;  
        # 最大长连数  
        keepalive 32;  
    }  
    # 配置lua依赖库地址  
    lua_package_path "/opt/openresty/lualib/kafka/?.lua;;";  
   
    server {  
        listen       80;  
        server_name  localhost;  
        location /favicon.ico {  
            root   html;  
                index  index.html index.htm;  
        }  
        location / {  
            proxy_connect_timeout 8;  
            proxy_send_timeout 8;  
            proxy_read_timeout 8;  
            proxy_buffer_size 4k;  
            proxy_buffers 512 8k;  
            proxy_busy_buffers_size 8k;  
            proxy_temp_file_write_size 64k;  
            proxy_next_upstream http_500 http_502  http_503 http_504  error timeout invalid_header;  
            root   html;  
            index  index.html index.htm;  
            proxy_pass http://rc;  
            proxy_http_version 1.1;  
            proxy_set_header Connection "";  
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;  
            # 使用log_by_lua 包含lua代码,因为log_by_lua指令运行在请求最后且不影响proxy_pass机制  
            log_by_lua '  
                -- 引入lua所有api  
                local cjson = require "cjson"  
                local producer = require "resty.kafka.producer"  
                -- 定义kafka broker地址，ip需要和kafka的host.name配置一致  
                local broker_list = {  
                    { host = "10.10.78.52", port = 9092 },  
                }  
                -- 定义json便于日志数据整理收集  
                local log_json = {}  
                log_json["uri"]=ngx.var.uri  
                log_json["args"]=ngx.var.args  
                log_json["host"]=ngx.var.host  
                log_json["request_body"]=ngx.var.request_body  
                log_json["remote_addr"] = ngx.var.remote_addr  
                log_json["remote_user"] = ngx.var.remote_user  
                log_json["time_local"] = ngx.var.time_local  
                log_json["status"] = ngx.var.status  
                log_json["body_bytes_sent"] = ngx.var.body_bytes_sent  
                log_json["http_referer"] = ngx.var.http_referer  
                log_json["http_user_agent"] = ngx.var.http_user_agent  
                log_json["http_x_forwarded_for"] = ngx.var.http_x_forwarded_for  
                log_json["upstream_response_time"] = ngx.var.upstream_response_time  
                log_json["request_time"] = ngx.var.request_time  
                -- 转换json为字符串  
                local message = cjson.encode(log_json);  
                -- 定义kafka异步生产者  
                local bp = producer:new(broker_list, { producer_type = "async" })  
                -- 发送日志消息,send第二个参数key,用于kafka路由控制:  
                -- key为nill(空)时，一段时间向同一partition写入数据  
                -- 指定key，按照key的hash写入到对应的partition  
                local ok, err = bp:send("test1", nil, message)  
   
                if not ok then  
                    ngx.log(ngx.ERR, "kafka send err:", err)  
                    return  
                end  
            ';  
        }  
        error_page   500 502 503 504  /50x.html;  
        location = /50x.html {  
            root   html;  
        }  
    }  
}  
```
#### 六:检测&运行
```
# 检测配置,只检测nginx配置是否正确，lua错误日志在nginx的error.log文件中  
./nginx -t /opt/openresty/nginx/conf/nginx.conf  
# 启动  
./nginx -c /opt/openresty/nginx/conf/nginx.conf  
# 重启  
./nginx -s reload  
```
#### 七:测试
```
http://10.10.78.52/m/personal/AC8E3BC7-6130-447B-A9D6-DF11CB74C3EF/rc/v1?passport=23456@qq.sohu.com&page=2&size=10
```
