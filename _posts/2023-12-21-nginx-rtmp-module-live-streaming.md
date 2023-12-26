---
layout: post
date: 2023-12-21
title: "nginx-rtmp 모듈 활용 실시간 스트리밍 구현"
published: true
lang: ko
excerpt: 실시간 스트리밍 기술에 대해 소개하고 스트리밍을 웹페이지에 송출하는 서버와 화면을 구현해 봅니다.
tags: nginx stream
author: seounghyun
---
## COVID-19 와 Live Stream
지난 몇년간 팬데믹으로 인해 사회적 거리두기와 국제 이동 제한이 강조되면서, 실시간 스트리밍 서비스가 주목받게 되었습니다. 온라인 활동이 급증함에 따라 비대면 서비스의 필요성이 부각되었고, 이로 인해 온라인 강의, 원격 회의, 영화 및 애니메이션 스트리밍 서비스등의 수요가 크게 증가하면서 이러한 활동들을 지원하는 실시간 스트리밍 기술이 필수적으로 대두되었습니다.  

실시간 스트리밍은 정보의 신속한 전달과 현장 소식을 실시간으로 공유하는 데 효과적이었습니다. Covid-19 관련 뉴스, 정부의 브리핑, 전문가 강연 등이 실시간으로 스트리밍되면서 대중들은 신뢰할 수 있는 정보에 보다 쉽게 접근할 수 있게 되었습니다.  

음악 콘서트, 스포츠 이벤트, 예술 공연과 같은 대규모 모임이 불가능한 상황에서는 이를 대체할 가상 이벤트 및 콘텐츠에 대한 수요가 증가하였습니다. 이에 따라 실시간으로 이벤트를 중계하거나, 아티스트들이 직접 집에서 콘서트를 열어 팬들과 소통하는 등의 노력이 진행되었습니다.  

실시간 스트리밍 기술은 다양한 플랫폼에서 쉽게 적용되며(특히 모바일), HTTP 기반의 프로토콜인 HLS와 MPEG-DASH를 비롯하여 다양한 전송 방식을 지원합니다. 최근 부상하고 있는 P2P기반 기술 WebRTC도 있습니다. 이번 게시글에서는 실시간 스트리밍 기술의 주요 구성 요소에 대한 간략한 소개와 더불어, 직접 스트리밍 서버를 구현하는 방법을 소개하겠습니다.

## 실시간 스트리밍 서비스의 구조
원본 영상이 시청자(클라이언트)에게 전달되기 까지는 아래와 같은 흐름을 거치게 됩니다.  
```
원본영상 -> 인코더 -> 미디어서버 -> 전송서버 -> 플레이어 -> 클라이언트
```
각 단계별로 수행하는 역할은 다음과 같습니다.
1. 원본영상 (Source Video)
   - 실제 비디오 또는 오디오 콘텐츠의 원본입니다.
   - 원본 영상은 인코딩 과정을 거쳐 다양한 형식으로 변환됩니다.
2. 인코더 (Encoder)
   - 원본 영상을 다양한 해상도 및 비트율로 변환하는 역할을 합니다.
   - 주로 H.264, H.265과 같은 비디오 코덱을 사용하여 압축 및 인코딩을 수행합니다.
3. 미디어 서버 (Media Server)
   - 인코딩된 미디어를 저장하고, 클라이언트에게 전송하는 역할을 합니다.
   - 주로 HTTP Live Streaming (HLS), Real-Time Messaging Protocol (RTMP), MPEG-DASH와 같은 프로토콜을 사용하여 클라이언트에게 미디어를 제공합니다.
4. 전송서버 (Content Delivery Network)
   - 분산된 서버 네트워크를 통해 콘텐츠를 효율적으로 전송하는 역할을 합니다.
   - 클라이언트가 위치한 지역에 가장 가까운 서버에서 콘텐츠를 제공하여 전송 속도를 향상시킵니다.
5. 플레이어 (Player)
   - 클라이언트 디바이스에서 동작하는 응용프로그램 또는 웹 브라우저 등을 의미합니다.
   - 미디어 서버 또는 CDN로부터 받은 스트림을 디코딩하고, 화면에 플레이어를 통해 출력합니다.
6. 클라이언트  (Client)
   - 스트리밍 서비스를 이용하는 사용자 또는 디바이스를 나타냅니다.
   - 플레이어를 통해 실시간으로 미디어 콘텐츠를 시청하거나 듣습니다.

![alt]({{ "/assets/2023-12-21-nginx-rtmp-module-live-streaming/2023-12-21-nginx-rtmp-module-live-streaming-01.svg" | absolute_url}}){: .center-image }

그림. Microsoft Azure live stream architecture
{: style="text-align: center; font-style: italic;"}

인코더는 원본영상을 정해진 방식으로 압축 및 인코딩하고 컨텐츠를 미디어 서버로 전달하는 역할을 합니다. 인코더에서 미디어 서버로 컨텐츠를 전송하는 것을 송출이라고 말합니다. 미디어 서버는 실시간 스트리밍의 핵심적인 역할을 담당합니다. 인코더가 보내준 영상을 여러 가지 화질 및 비트레이트로 변환합니다. 그리고 변환된 영상을 표준 프로토콜(HLS등)로 변환합니다.(이걸 트랜스믹싱 이라고 합니다)  

## HLS

![alt]({{ "/assets/2023-12-21-nginx-rtmp-module-live-streaming/2023-12-21-nginx-rtmp-module-live-streaming-02.png" | absolute_url}}){: .center-image }

그림. Developer Apple HTTP Live Streaming Overview
{: style="text-align: center; font-style: italic;"}

HLS(Http Live Streaming)은 Apple에서 개발한 비디오 스트리밍 프로토콜입니다. HLS는 비디오 파일을 다운로드할 수 있는 Http 파일 조각으로 나누고 Http 프로토콜을 사용해서 전송합니다. HLS는 동영상을 세그먼트로 분할하고, 클라이언트는 세그먼트를 다운로드하여 연속적으로 재생함으로써 스트리밍 영상을 시청할 수 있습니다. 영상재생을 위한 메타정보를 담고 있는 m3u8 마스터 플레이리스트 파일과 잘게 쪼개진 미디어 파일인 ts파일로 구성되어있습니다.  

### HLS의 장점
- 호환성 : HTTP 프로토콜을 지원하는 다양한 장치에서 스트리밍을 할 수 있습니다.
- 부드러운 재생 : ABR 기능으로 무중단 스트리밍중에 품질을 유지시킵니다.
- 보안 : Flash 보다 더 안전한 프로토콜입니다.

### HLS의 단점
- 높은 대기 시간 : HLS는 대기 시간이 길어서 20초~30초의 최대 지연이 있을 수 있습니다.
- 인터넷 속도 : HLS는 지연 시간이 상대적으로 높기 때문에 비디오 게임이나 스포츠 방송같은 라이브 스트림에서는 추가적인 튜닝이 필요할 수 있습니다.

## HLS 스트리밍 구현
그러면 이제 간단한 실시간 스트리밍 테스트를 해보도록 하겠습니다. 우선 OBS를 설치합니다. OBS는 인터넷 방송 및 동영상 캡처를 지원하는 오픈소스 소프트웨어 입니다. 웹캠이나 사용자 화면, 또는 이미지등을 실시간으로 방송 송출할 수 있게 해줍니다. OBS를 사용해서 사용자 화면을 캡처한뒤에 RTMP 프로토콜로 인코딩하고, nginx-rtmp 미디어 서버를 구축해서 플레이어에서 재생 가능한 HLS 스트리밍으로 트랜스믹싱 후에 웹페이지에서 표시해보겠습니다.

![alt]({{ "/assets/2023-12-21-nginx-rtmp-module-live-streaming/2023-12-21-nginx-rtmp-module-live-streaming-03.png" | absolute_url}}){: .center-image width="70%"}

### nginx-rtmp 설치
```
sudo yum install pcre pcre-devel openssl openssl-devel zlib zlib-devel -y
sudo yum groupinstall "Development Tools" -y
```
nginx 컴파일 설치를 위한 의존성 도구들을 설치합니다.  

```
mkdir /usr/local/nginx-rtmp
cd /usr/local/nginx-rtmp

wget https://nginx.org/download/nginx-1.14.0.tar.gz
wget https://github.com/arut/nginx-rtmp-module/archive/master.zip
tar -zxvf nginx-1.14.0.tar.gz
unzip master.zip
```
이제 nginx, nginx-rtmp 모듈을 다운받고 압축을 풀어줍니다.
```
cd nginx-1.14.0

sudo ./configure --prefix=/etc/nginx --sbin-path=/usr/sbin/nginx --conf-path=/etc/nginx/nginx.conf --pid-path=/var/run/nginx.pid --error-log-path=/var/log/nginx/error.log --http-log-path=/var/log/nginx/access.log --with-http_ssl_module --add-module=../nginx-rtmp-module-master --user=nginx --group=nginx

sudo make

sudo make install
```
nginx와 rtmp 모듈을 컴파일 설치 하겠습니다. --add-module 설정에 nginx-rtmp모듈 설치위치를 설정합니다.

```
sudo useradd --shell /sbin/nologin nginx
```
nginx 사용자를 등록하고 서비스에 등록해서 실행하겠습니다.

```
sudo vi /etc/init.d/nginx
# 아래 내용을 복사해서 편집기에 붙여넣고 저장합니다.

#!/bin/sh
#
# nginx - this script starts and stops the nginx daemon
#
# chkconfig:   - 85 15
# description:  NGINX is an HTTP(S) server, HTTP(S) reverse \
#               proxy and IMAP/POP3 proxy server
# processname: nginx
# config:      /etc/nginx/nginx.conf
# config:      /etc/sysconfig/nginx
# pidfile:     /var/run/nginx.pid

# Source function library.
. /etc/rc.d/init.d/functions

# Source networking configuration.
. /etc/sysconfig/network

# Check that networking is up.
[ "$NETWORKING" = "no" ] && exit 0

nginx="/usr/sbin/nginx"
prog=$(basename $nginx)

NGINX_CONF_FILE="/etc/nginx/nginx.conf"

[ -f /etc/sysconfig/nginx ] && . /etc/sysconfig/nginx

lockfile=/var/lock/subsys/nginx

make_dirs() {
   # make required directories
   user=`$nginx -V 2>&1 | grep "configure arguments:.*--user=" | sed 's/[^*]*--user=\([^ ]*\).*/\1/g' -`
   if [ -n "$user" ]; then
      if [ -z "`grep $user /etc/passwd`" ]; then
         useradd -M -s /bin/nologin $user
      fi
      options=`$nginx -V 2>&1 | grep 'configure arguments:'`
      for opt in $options; do
          if [ `echo $opt | grep '.*-temp-path'` ]; then
              value=`echo $opt | cut -d "=" -f 2`
              if [ ! -d "$value" ]; then
                  # echo "creating" $value
                  mkdir -p $value && chown -R $user $value
              fi
          fi
       done
    fi
}

start() {
    [ -x $nginx ] || exit 5
    [ -f $NGINX_CONF_FILE ] || exit 6
    make_dirs
    echo -n $"Starting $prog: "
    daemon $nginx -c $NGINX_CONF_FILE
    retval=$?
    echo
    [ $retval -eq 0 ] && touch $lockfile
    return $retval
}

stop() {
    echo -n $"Stopping $prog: "
    killproc $prog -QUIT
    retval=$?
    echo
    [ $retval -eq 0 ] && rm -f $lockfile
    return $retval
}

restart() {
    configtest || return $?
    stop
    sleep 1
    start
}

reload() {
    configtest || return $?
    echo -n $"Reloading $prog: "
    killproc $nginx -HUP
    RETVAL=$?
    echo
}

force_reload() {
    restart
}

configtest() {
  $nginx -t -c $NGINX_CONF_FILE
}

rh_status() {
    status $prog
}

rh_status_q() {
    rh_status >/dev/null 2>&1
}

case "$1" in
    start)
        rh_status_q && exit 0
        $1
        ;;
    stop)
        rh_status_q || exit 0
        $1
        ;;
    restart|configtest)
        $1
        ;;
    reload)
        rh_status_q || exit 7
        $1
        ;;
    force-reload)
        force_reload
        ;;
    status)
        rh_status
        ;;
    condrestart|try-restart)
        rh_status_q || exit 0
            ;;
    *)
        echo $"Usage: $0 {start|stop|status|restart|condrestart|try-restart|reload|force-reload|configtest}"
        exit 2
esac
```
nginx 서비스 실행을 위해 실행파일을 작성합니다.
```
sudo chmod +x /etc/init.d/nginx
```
서비스 실행파일 권한 설정을 변경합니다.
```
vi /etc/nginx/nginx.conf

# 아래 내용으로 설정파일을 덮어씌웁니다.

#user  nobody;
worker_processes  auto;

events {
    worker_connections  1024;
}

rtmp {

    server {

        listen 1935;

        chunk_size 4000;

        application hls {
            live on;
            hls on;
            hls_path /tmp/hls;
            allow publish 127.0.0.1;
            allow publish 172.16.0.0/12;
            deny publish all;
            allow play all;
            hls_fragment 600ms;
            hls_playlist_length 5s;
        }

    }
}

# HTTP can be used for accessing RTMP stats
http {

    server {

        listen      8080;

        location /hls {
            
            # Disable cache
            add_header Cache-Control no-cache;

            # CORS setup
            add_header 'Access-Control-Allow-Origin' '*' always;
            add_header 'Access-Control-Expose-Headers' 'Content-Length';

            # allow CORS preflight requests
            if ($request_method = 'OPTIONS') {
                add_header 'Access-Control-Allow-Origin' '*';
                add_header 'Access-Control-Max-Age' 1728000;
                add_header 'Content-Type' 'text/plain charset=UTF-8';
                add_header 'Content-Length' 0;
                return 204;
            }
	    # Serve HLS fragments
            types {
                application/vnd.apple.mpegurl m3u8;
                video/mp2t ts;
            }
            root /tmp;
            add_header Cache-Control no-cache;
        }
    }
}
```
다음으로 nginx 설정파일을 변경합니다. rtmp 프로토콜로 송출되는 스트림을 hls 스트림으로 인코딩하는 미디어 서버 역할을 합니다.

```
servcie nginx restart

# 아래와 같이 실행됩니다.
Restarting nginx (via systemctl):                          [  OK  ]
```

### OBS에서 방송 송출 설정 및 화면 표시
![alt]({{ "/assets/2023-12-21-nginx-rtmp-module-live-streaming/2023-12-21-nginx-rtmp-module-live-streaming-04.png" | absolute_url}}){: .center-image width="70%"}

OBS 프로그램에서 파일 > 설정 > 방송으로 들어갑니다. 서버를 nginx-rtmp가 실행되고 있는 주소로 세팅하고, 미디어 서버로 스트림을 송출합니다. 저의 경우는 내부 가상 컴퓨터 환경에 설치하고 실행했기 때문에, 내부ip를 세팅했습니다. 스트림 키의 경우에는 아무렇게나 설정해도 상관없습니다.  

이제 제어 탭의 방송시작 버튼을 클릭하면 파란색의 방송 중단으로 바뀌면서 송출이 시작됩니다. 만약 에러가 발생한다면 nginx 서버의 포트나 방화벽 설정및 네트워크 세팅을 확인하거나, 정상적으로 nginx 서버가 실행중인지 확인해야합니다.  

```html
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="UTF-8">
        <title>HLS Streaming</title>
        <script src="https://cdn.jsdelivr.net/npm/hls.js@latest"></script>
    </head>
    <body>
        <video id="video" width="640" controls autoplay></video>
    </body>
    <script>
        var video = document.getElementById('video');
        var videoSrc = 'http://172.21.173.218:8080/hls/steamkey.m3u8';
        //
        // First check for native browser HLS support
        //
        if (video.canPlayType('application/vnd.apple.mpegurl')) {
            video.src = videoSrc;
            //
            // If no native HLS support, check if HLS.js is supported
            //
        } else if (Hls.isSupported()) {
            var hls = new Hls();
            hls.loadSource(videoSrc);
            hls.attachMedia(video);
        }
    </script>
</html>
```
hls.js 라이브러리를 사용해서 웹페이지에서 스트리밍중인 화면을 video태그에 소스로 추가합니다. 이때 m3u8 파일의 이름은 OBS에서 스트림 키로 설정했던 이름으로 해야합니다.

![alt]({{ "/assets/2023-12-21-nginx-rtmp-module-live-streaming/2023-12-21-nginx-rtmp-module-live-streaming-05.png" | absolute_url}}){: .center-image width="70%"}

로컬 사용자 기준 html 웹페이지를 띄운 화면에 video 태그에서 실시간 스트리밍을 재생해서 확인할 수 있습니다.