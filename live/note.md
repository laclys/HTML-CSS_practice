PC端/IOS/Android -> 压缩编码过程（H.264/ACC）-> 字幕叠加（视频本身处理）-> 推流 -> CDN -> 直播（终端显示直播流）

直播协议： HLS协议 RTMP协议 HTTP-FLV协议


HLS协议：（apple推出的只支持safari，格式ts）有延迟 和切片有关

video -> M3U8(索引文件,对应数个ts格式文件片段)，m3u8文件只是当前时间段的索引,segment.ts都会有个时长,在segment总时长之前,video会重新向服务器请求m3u8文件，这是浏览器自身的行为，此时，服务器会返回新的m3u8文件，更新后的segment.ts文件将会改变(m3u8文件结构并不一定全是由ts文件组成，有可能会嵌套一层m3u8文件)

对于每个ts文件来说，第一个ts文件会有一个PAT的包，解析时，这个pat的包会告诉你去找一个PMT的包，PMT会告诉你后面要解析到的ts文件中，哪些是视频哪些是音频


RTMP协议（Real Time Messaging Protocol）（实时性比较好不过使用起来比较复杂,传输的视频格式为flv）

HTTP-FLV结合了HLS的优点,采用HTTP请求,又结合了RTMP低延时的特性,因为中间建立了FLV的长连接,中间通信过程相对RTMP协议就不再复杂

HTTP-FLV优点（相对RTMP）：
1.可以在一定程度上避免防火墙干扰（有些机房只允许80端口通过）
2.可以很好的兼容HTTP 302跳转
3.可以使用HTTPS
4很好的支持移动端


直播源的制作
- Nginx + ffmpeg

1.安装Nginx

换一个github项目下nginx

$ brew tap denji/nginx

$ brew install nginx-full --with-upload-module

验证：http://localhost:8080/

2.安装ffmpeg

brew install ffmpeg

3.配置Nginx
cd /usr/local/etc/nginx/
nginx.conf
```
rtmp{
    server {
        listen 1935;
        chunk_size 4000;

        # RTMP 直播流配置
        application rtmplive {
            live on;
            max_connections 1024;
        }

        # hls 直播流配置
        application hls{
            live on;
            hls on;
            hls_path /usr/local/var/www/hls;
            hls_fragment 5s;
        }
    }
}
```
```
        location /hls {
            types{
                application/vnd.apple.mpegurl m3u8;
                video/mp2t ts;
            }
            root /usr/local/var/www;
            add_header Cache-Control no-cache;
        }
```
4.准备视频
5.利用ffmpeg推流


RTMP 直播：
ffmpeg -re -i test2.mp4 -vcodec libx264 -acodec aac -f flv rtmp://localhost:1935/rtmplive/rtmp


使用 VLC 打开 rtmp://localhost:1935/rtmplive/rtmp

hls 直播：
ffmpeg -re -i test2.mp4 -vcodec libx264 -acodec aac -f flv rtmp://localhost:1935/hls/stream

safari http://localhost:8080/hls/stream.m3u8