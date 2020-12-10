FFmpeg + nginx-http-flv-module + flv.js 实现视频流播放 包括了 HLS (m3us flv dash) 三种模式 推荐 flv

前期准备
下载VLC（用于测试视频流是否可以播放）：https://www.videolan.org/
下载nginx包：https://nginx.org/download/nginx-1.16.1.tar.gz
下载nginx-http-flv-module 模块包：https://github.com/winshining/nginx-http-flv-module
下载FFmpeg：http://www.ffmpeg.org/releases/ffmpeg-4.2.2.tar.bz2
下载yasm（安装FFmpeg可能会报错，所以需要安装yasm）：http://www.tortall.net/projects/yasm/releases/yasm-1.3.0.tar.gz
下载x264（视频转码需要用到）：https://git.videolan.org/git/x264.git
下载nasm（x264编译时需要安装nasm）：https://www.nasm.us/pub/nasm/releasebuilds/2.14.02/nasm-2.14.02.tar.bz2

一、搭建环境
安装nginx
在 /user/local 下创建 nginx 文件夹

mkdir /usr/local/nginx

安装依赖项

yum -y install unzip
yum -y install gcc-c++ 
yum -y install pcre pcre-devel  
yum -y install zlib zlib-devel 
yum -y install openssl openssl-devel

将 /user/local 下面的 nginx-http-flv-module-master.zip 解压到 /usr/local/nginx 下面

cd /usr/local
unzip nginx-http-flv-module-master.zip  //解压文件
cp -r /usr/local/nginx-http-flv-module-master /usr/local/nginx/nginx-http-flv-module  //解压文件复制到/usr/local/nginx 目

将 nginx-http-flv-module 模板添加到 nginx 中，生成 make 文件并安装 nginx

cd /usr/local
tar -zxvf nginx-1.16.1.tar.gz
cd nginx-1.16.1/
./configure --prefix=/usr/local/nginx  --add-module=/usr/local/nginx/nginx-http-flv-module
make 
make install
ln -s /usr/local/nginx/sbin/nginx /usr/local/bin/  //配置nginx为全局变量
nginx -v  //查看nginx是否安装成功

二、修改配置文件
1.修改配置文件nginx.conf
cd /usr/local/nginx/conf
vi nginx.conf

worker_processes  1; #Nginx开启1个子进程，子进程个数最好跟CPU的核心数一样
#worker_processes  auto; #from versions 1.3.8 and 1.2.5

#worker_cpu_affinity  0001 0010 0100 1000; #CPU的mask，子进程使用它来绑定CPU核心，避免进程切换造成性能损失
#worker_cpu_affinity  auto; #from version 1.9.10

error_log logs/error.log error; #错误日志位置和日志级别，如果使用默认编译选项，位置为/usr/local/nginx/logs/error.log，error表示只打印错误日志

events {
    worker_connections  4096; #Nginx处理的最大连接数
}

http {
    include             mime.types;
    default_type    application/octet-stream;
    sendfile            on;
    keepalive_timeout  65;

    server {
        listen       80; #Nginx监听的HTTP请求端口
        server_name  localhost;
        location / {
            root    html; #HTTP请求URL映射到服务器的位置
            index  index.html index.htm; #HTTP请求优先请求的文件
        } 

        location /live {
            flv_live on; #当HTTP请求以/live结尾，匹配这儿，这个选项表示开启了flv直播播放功能
            chunked_transfer_encoding  on; #HTTP协议开启Transfer-Encoding: chunked;方式回复

            add_header 'Access-Control-Allow-Origin' '*'; #添加额外的HTTP头
            add_header 'Access-Control-Allow-Credentials' 'true'; #添加额外的HTTP头
        }

        location /stat {
            rtmp_stat all;
            rtmp_stat_stylesheet stat.xsl;
        }

        location /stat.xsl {
            root  /usr/local/nginx/module/nginx-http-flv-module;
        }

        location /control {
            rtmp_control all;
        }
        
        error_page   500 502 503 504  /50x.html; #如果遇到这些HTTP请求错误，Nginx返回50x.html的内容
        location = /50x.html {
            root   html;
        }
    }
}

rtmp_auto_push on; #因为Nginx可能开启多个子进程，这个选项表示推流时，媒体流会发布到多个子进程
rtmp_auto_push_reconnect 1s;
rtmp_socket_dir /tmp; #多个子进程情况下，推流时，最开始只有一个子进程在竞争中接收到数据，然后它再relay给其他子进程，他们之间通过unix domain socket传输数据，这个选项表示unix

rtmp {
    out_queue                4096;
    out_cork                   8;
    max_streams            128; #Nginx能接受的最大的推流数
    timeout                    15s;
    drop_idle_publisher 15s;

    log_interval 5s; 
    log_size     1m; 

    server {
        listen 1935; #Nginx监听的RTMP推流/拉流端口，可以省略，默认监听1935
        server_name 127.0.0.1;

        application myapp {
            live on; #当推流时，RTMP路径中的APP（RTMP中一个概念）匹配myapp时，开启直播
            record off; #不记录视频
            gop_cache on;  #开启GOP（Group of Picture）缓存，播放器解码时，收到一个完整的GOP才会开始播放，这个是减少播放延迟的选项
        }
    }
}


测试nginx是否好用

nginx -t //检查nginx
nginx //启动
ps -ef | grep nginx //查看nginx进程
浏览器输入地址是否可以访问成功

2.安装FFmpeg
安装前期准备下载的nasm

cd /usr/local
tar -jxvf nasm-2.14.02.tar.bz2  //如果解压不了的话安装一下bzip2（ yum install -y bzip2 ）
cd nasm-2.14.02/
./configure
make
make install

安装前期准备下载的yasm

cd /usr/local
tar -zxvf yasm-1.3.0.tar.gz
cd yasm-1.3.0/
./configure
make && make install

安装x264（RTSP视频流转码时需要x264）

cd /usr/local/x264
./configure --enable-static --enable-shared
make && make install

安装FFmpeg

cd /usr/local
tar -jxvf ffmpeg-4.2.2.tar.bz2
cd ffmpeg-4.2.2/
./configure --prefix=/usr/local/ffmpeg  --enable-gpl --enable-libx264
make && make install  （FFmpeg编译时间比较长）
ln -s /usr/local/ffmpeg/bin/ffmpeg /usr/local/bin  //配置ffmpeg全局环境变量
ffmpeg -version  //查看ffmpeg是否安装成功
直接运行FFmpeg可能报如下错误（重点）

ffmpeg: error while loading shared libraries: libx264.so.157: cannot open shared object file: No such file or directory
解决：
vi /etc/ld.so.conf
添加libx264.so文件位置
我的是在：/usr/local/lib/ 加入到 ld.so.conf
重载配置:ldconfig


3.推流
ffmpeg -re -rtsp_transport tcp -i "rtsp://admin:Biao456813@192.168.153.194:554/h264/ch1/main/av_stream" -f flv -vcodec libx264 -vprofile baseline -acodec aac -tune zerolatency -preset ultrafast -ar 44100 -strict -2 -ac 1 -f flv -s 1980x1080-q 10 "rtmp://192.168.153.204:1935/myapp/home"
