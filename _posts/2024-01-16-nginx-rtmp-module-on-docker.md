---
layout: post
date: 2024-01-16
title: "Docker로 nginx 서버 실행하기"
published: true
lang: ko
excerpt: 윈도우 환경에서 Docker Desktop을 사용해서 nginx 스트리밍 서버를 실행시키는 방법
tags: nginx stream docker
author: seounghyun
---
## 컨테이너 기술과 Docker Desktop

<pre>
<em>컨테이너는 실행에 필요한 모든 파일을 포함한 전체 실행(runtime) 환경에서 애플리케이션을 패키징하고 격리할 수 있는 기술입니다.   
이를 통해 전체 기능을 유지하면서 컨테이너화된 애플리케이션을 환경(개발, 테스트, 프로덕션 환경 등) 간에 쉽게 이동할 수 있습니다. - Red Hat 컨테이너의 이해</em>
</pre>
지난 게시글 [nginx-rtmp 모듈 활용 실시간 스트리밍][2023-12-21-nginx-rtmp-module-live-streaming] 구현 에서는 Hyper-V를 사용해 가상컴퓨터를 구성해서 그 위에 리눅스OS를 올려 스트리밍 서버를 구성했습니다. 하지만 이렇게 구성할 경우 가상컴퓨터 ip를 내부의 가상ip를 자동으로 할당하고, 서버 테스트간에 항상 Hyper-V를 실행해야 하는 번거로움이 있었습니다.   
그래서 이번엔 컨테이너 기술인 Docker를 사용하여 윈도우 환경에서 로컬 환경에 컨테이너를 구축하고, nginx 스트리밍 서버를 실행시켜 보겠습니다. 
## Docker Desktop 설치
도커는 기본적으로 리눅스 컨테이너라서 리눅스 OS 기반으로 동작하지만, 이런 불편함을 해소하기 위해 Docker Desktop을 공개해 Windows와 Mac 환경에서 도커를 손쉽게 사용할 수 있도록 지원하고 있습니다.   
Docker Desktop은 기본적으로 자체 가상화 기능을 사용하기 때문에 사용하는 Windows OS에서 Hyper-V 기능을 지원해야합니다. 즉, Windows Pro에디션 이상에서만 사용할 수 있었습니다. 하지만 2020년 즈음 Windows업데이트가 진행되면서 WSL2(Windows Subsystem for Linux)를 Home버전에서 지원하게 되면서 Docker Desktop을 WSL2 기반으로 Home 버전에서도 사용하는게 가능해졌습니다.   

- Windows 10/11 Professional / Education / Enterprise 에디션
    - WSL2 기반 Docker Engine 사용 가능
    - Hyper-V 기반 Docker Engine 사용 가능
- Windows 10/11 Home 에디션
    - WSL2 기반 Docker Engine 사용 가능

파워셸에서 아래 명령어로 WSL 설치 및 기본값 변경을 실행해줍니다.

```
$ wsl --install
```
```
$ wsl --set-default-version 2
```

https://www.docker.com/products/docker-desktop/   
다음으로 도커 데스크탑 인스톨러 for Windows 파일을 다운로드 후에 실행시킵니다. 복잡한 부분은 없고 클릭 몇번으로 설치가 가능합니다. 만약 본인이 사용하는 Windows가 Home에디션인 경우 Configuration에서 'Use WSL 2 instead of Hyper-V (recommended)'를 체크해주면 됩니다. 몇분뒤 설치가 완료되면 시스템 리부트가 필요하고 이후 Docker Desktop을 실행할 수 있습니다.

![alt]({{ "/assets/2024-01-16-nginx-rtmp-module-on-docker/2024-01-16-nginx-rtmp-module-on-docker-01.png" | absolute_url}}){: .center-image }

```
$ docker version //CDM 프롬프트에서 도커 버전 확인

Client:
 Cloud integration: v1.0.35+desktop.5
 Version:           24.0.7
 API version:       1.43
 Go version:        go1.20.10
 Git commit:        afdd53b
 Built:             Thu Oct 26 09:08:44 2023
 OS/Arch:           windows/amd64
 Context:           default

Server: Docker Desktop 4.26.1 (131620)
 Engine:
  Version:          24.0.7
  API version:      1.43 (minimum version 1.12)
  Go version:       go1.20.10
  Git commit:       311b9ff
  Built:            Thu Oct 26 09:08:02 2023
  OS/Arch:          linux/amd64
  Experimental:     false
 containerd:
  Version:          1.6.25
  GitCommit:        d8f198a4ed8892c764191ef7b3b06d8a2eeb5c7f
 runc:
  Version:          1.1.10
  GitCommit:        v1.1.10-0-g18a0cb0
 docker-init:
  Version:          0.19.0
  GitCommit:        de40ad0
```
## Docker 이미지 빌드 및 컨테이너 실행
도커 데스크탑이 설치가 완료됐으면 이제 컨테이너로 올릴 이미지가 필요합니다. 도커 이미지는 도커 어플리케이션 실행에 필요한 독립적인 환경을 포함하는 일종의 스냅샷 템플릿입니다. 소스 코드, 라이브러리, 종속환경, 응용 프로그램등을 실행하는 파일을 포함하는 변경 불가능한 파일입니다.   
도커 이미지 파일은 Dockerfile을 작성해서 빌드하는 과정으로 생성합니다. 이번에는 따로 Dockerfile 작성에 필요한 문법이나 코드를 자세하게 알아보지는 않고, 미리 제공하는 파일을 사용하겠습니다.   

```
FROM alpine:3.13.4 as builder
RUN apk add --update build-base git bash gcc make g++ zlib-dev linux-headers pcre-dev openssl-dev
RUN git clone https://github.com/arut/nginx-rtmp-module.git && \
    git clone https://github.com/nginx/nginx.git
RUN cd nginx && ./auto/configure --add-module=../nginx-rtmp-module && make && make install


FROM alpine:3.13.4 as nginx
RUN apk add --update pcre ffmpeg
COPY --from=builder /usr/local/nginx /usr/local/nginx
ENTRYPOINT ["/usr/local/nginx/sbin/nginx"]

EXPOSE 1935
EXPOSE 8080

CMD ["-g", "daemon off;"]
```
위와 같은 내용으로 'Dockerfile'이라는 파일을 작성후에 원하는 경로에 저장합니다.(확장자는 필요없습니다) 도커파일 내용을 간략하게 설명하자면, 경량 리눅스 OS인 알파인을 베이스로 구축하여 nginx와 nginx-rtmp모듈을 설치하고, 컴파일후에 nginx를 실행시키겠다- 라는 내용을 담고있습니다.   
명령 프롬프트를 열어 도커파일이 있는 경로로 이동한뒤에, 도커 빌드 명령어를 실행합니다.
```
D:\nginx>docker build -t nginx-rtmp .

[+] Building 3.0s (10/10) FINISHED                                                                       docker:default
 => [internal] load build definition from Dockerfile                                                               0.0s
 => => transferring dockerfile: 598B                                                                               0.0s
 => [internal] load .dockerignore                                                                                  0.0s
 => => transferring context: 2B                                                                                    0.0s
 => [internal] load metadata for docker.io/library/alpine:3.13.4                                                   2.8s
 => [builder 1/4] FROM docker.io/library/alpine:3.13.4@sha256:ec14c7992a97fc11425907e908340c6c3d6ff602f5f13d899e6  0.0s
 => CACHED [nginx 2/3] RUN apk add --update pcre ffmpeg                                                            0.0s
 => CACHED [builder 2/4] RUN apk add --update build-base git bash gcc make g++ zlib-dev linux-headers pcre-dev op  0.0s
 => CACHED [builder 3/4] RUN git clone https://github.com/arut/nginx-rtmp-module.git &&     git clone https://git  0.0s
 => CACHED [builder 4/4] RUN cd nginx && ./auto/configure --add-module=../nginx-rtmp-module && make && make insta  0.0s
 => CACHED [nginx 3/3] COPY --from=builder /usr/local/nginx /usr/local/nginx                                       0.0s
 => exporting to image                                                                                             0.0s
 => => exporting layers                                                                                            0.0s
 => => writing image sha256:cf2bbab658ac8a8aa751ceca532546cc559a7017c963f9d2724350c2e9a8f919                       0.0s
 => => naming to docker.io/library/nginx-rtmp                                                                      0.0s

View build details: docker-desktop://dashboard/build/default/default/zg6m8balnk3wg394mg2m3a50q

What's Next?
  View a summary of image vulnerabilities and recommendations → docker scout quickview
```

도커 데스크탑 화면에서 Builds메뉴를 클릭하면 완료된 빌드 결과와 로그를 확인할 수 있고, Images메뉴에서 도커 파일로 생성한 이미지를 확인할 수 있습니다.

![alt]({{ "/assets/2024-01-16-nginx-rtmp-module-on-docker/2024-01-16-nginx-rtmp-module-on-docker-02.png" | absolute_url}}){: .center-image }

![alt]({{ "/assets/2024-01-16-nginx-rtmp-module-on-docker/2024-01-16-nginx-rtmp-module-on-docker-03.png" | absolute_url}}){: .center-image }

## Volume

![alt]({{ "/assets/2024-01-16-nginx-rtmp-module-on-docker/2024-01-16-nginx-rtmp-module-on-docker-04.png" | absolute_url}}){: .center-image }

이제 빌드한 이미지를 사용해서 컨테이너로 배포하고 서버를 실행시켜 보겠습니다. 도커 명령어로 실행시키거나, 도커 데스크탑에서 이미지를 클릭하고 Run 버튼으로 실행시킬 수도 있습니다. Containers 메뉴에서 실행중인 컨테이너 목록과 상태를 확인 할 수 있습니다.   

도커에서 컨테이너로 배포한 시스템은 휘발성이라서, 실행을 종료하면 저장했던 데이터나 로그가 전부 지워집니다. 기존에 스트리밍 서버로 사용하기 위해 작성했던 nginx.conf파일을 적용하기 위해, Volume을 사용하겠습니다. Volume은 도커에서 지원하는 마운트 기능으로, 도커 서버가 실행되는 로컬환경의 경로를 컨테이너의 가상 위치와 동기화시키는 기능입니다. 실행중인 서버의 설정파일을 변경해도 컨테이너를 다시 시작하면 설정한 파일이 전부 초기 이미지 환경으로 돌아가기 때문에, Volume을 설정해서 컨테이너를 실행시킬때마다 설정 파일을 적용시키겠습니다.   

```
D:\nginx>docker volume create nginx.conf
nginx.conf

D:\nginx>docker volume ls
DRIVER    VOLUME NAME
local     nginx.conf

D:\nginx>docker run -v nginx.conf:/usr/local/nginx/conf nginx-rtmp
```
도커 볼륨 명령어로 nginx.conf 볼륨을 생성한뒤에, nginx-rtmp 이미지의 /usr/local/nginx/conf 경로를 해당 볼륨에 마운트 시키면서 실행시키겠습니다. 정상적으로 볼륨이 마운트되면 컨테이너를 종료하고, 볼륨 명령어로 상세 정보를 확인 하거나 도커 데스크탑에서 실제 경로의 파일 정보를 확인 할 수 있습니다.

![alt]({{ "/assets/2024-01-16-nginx-rtmp-module-on-docker/2024-01-16-nginx-rtmp-module-on-docker-05.png" | absolute_url}}){: .center-image }

nginx.conf 파일을 열어서 안의 내용을 기존에 작성했던 nginx 설정 파일로 덮어씌우고, 저장하면 해당 볼륨을 사용할 준비가 끝났습니다.   

<details>
<summary>nginx.conf</summary>
<div markdown="1">

```
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
</div>
</details>


```
D:\nginx>docker run -d --name Streaming -v nginx.conf:/usr/local/nginx/conf/ -p 8080:8080 -p 1935:1935 nginx-rtmp
```
이제 도커 명령어로 볼륨을 사용해서 실행시킵니다.
- -d : 도커 컨테이너를 백그라운드로 실행시킵니다.
- --name : 도커 컨테이너 이름을 설정합니다.
- -v : 마운트 시킬 볼륨과 경로를 설정합니다.
- -p : 호스트와 바인딩시킬 포트를 설정합니다.
- nginx-rtmp : 사용할 이미지 이름입니다.

도커 데스크탑에서 실행중인 컨테이너의 내부 파일 구조를 쉽게 확인할 수 있습니다. Files탭에서 볼륨으로 마운트시킨 경로인 /usr/local/nginx/conf/로 가보면 VOLUME으로 표시되고 실제로 nginx.conf파일이 수정했던 내용으로 적용되어 컨테이너가 실행중인걸 확인할 수 있습니다.

![alt]({{ "/assets/2024-01-16-nginx-rtmp-module-on-docker/2024-01-16-nginx-rtmp-module-on-docker-06.png" | absolute_url}}){: .center-image }

이제 이전 게시글에서 테스트했던 OBS를 사용한 방송 송출및 플레이어 표시 테스트에서, 가상 ip 세팅 부분을 localhost로 세팅하고 실행하면 정상적으로 nginx서버가 스트리밍을 하는걸 확인할 수 있습니다. 가상컴퓨터 위에 OS 및 서버를 올리지않고, 도커 데스크탑을 사용해서 컨테이너 형태로 서버를 배포하여 운용하여 배포 및 관리의 편의성을 얻을수 있습니다.

[2023-12-21-nginx-rtmp-module-live-streaming]: https://cheonseounghyun.github.io/2023/12/21/nginx-rtmp-module-live-streaming.html