---
title: "Docker Nginx 그리고 Logrotate"
last_modified_at: 2021-01-11T22:00+00:00
header:
  show_overlay_excerpt: false
  overlay_image: /assets/images/post/2021/01/2021-01-11-title.jpg
  og_image: /assets/images/post/2021/01/2021-01-11-title.jpg
  overlay_filter: 0.6
  caption: "Photo by Priscilla Du Preez on Unsplash"
  
tags:
  - Docker
  - Server
category: #카테고리
  - Docker
toc: false
toc_label: "Table Of Contents"
toc_icon: "fal fa-list-alt"
toc_sticky: true
classes: wide
comments: true
---



docker로 nginx를 실행하고 logrotate 설정하는 방법을 확인해 보겠습니다. nginx alpine 이미지에는 logrotate가 없지만, 호스트에 logrotate가 있다면 docker volume을 통해서 log 파일을 logrotate를 실행할 수 있습니다.
이렇게 구성하게 되면 호스트가 변경되면 logrotate를 다시 설정해야 하는 번거로움이 생깁니다. 그래서 nginx docker 컨테이너 안에서 logrotate를 실행하는 구조로 만드는 것이 더 좋다고 생각합니다.

## Dockerfile 생성
logrotate가 포함된 nginx 이미지를 만들기 위해서 Dockerfile을 생성해보겠습니다.

### Logrotate
먼저 logrotate 설정 파일을 하나 준비합니다. `/logs/*.log`는 docker 컨테이너 안의 경로입니다.
옵션 정보는 검색하면 많이 나오기 때문에 생략합니다.

```
/logs/*.log {
    daily
    rotate 3
    missingok
    copytruncate
    dateext
    sharedscripts
    postrotate
        [ -s /run/nginx.pid ] && kill -USR1 `cat /run/nginx.pid`
    endscript
}
```

### nginx.conf
docker nginx에 있는 기본 conf을 참고해서 심플하게 생성한 nginx.conf 파일입니다. nginx실행시 `/logs` 디렉토리 밑에 로그 파일이 생성되도록 했습니다.
```nginx
user  nginx;
worker_processes  auto;

events {
    worker_connections  1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    keepalive_timeout  60;

    server {
      listen 80;

      access_log /logs/access.log;
      error_log /logs/error.log;

      location / {
        root   /usr/share/nginx/html;
        index  index.html;
      }
    }
}
```

### Dockerfile
nginx alpine에는 logrotate가 없기 때문에 docker 이미지 생성 시 먼저 설치하고 앞서 생성한 logrotate, nginx.conf 파일을 이미지에 포함시킵니다.
```dockerfile
FROM nginx:1.19.6-alpine

# Install logrotate
RUN apk add --no-cache logrotate

COPY ./nginx.conf /etc/nginx/nginx.conf
COPY ./nginx /etc/logrotate.d/

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Docker 이미지 생성
Dockerfile에 있는 경로에서 아래 커맨드를 입력하면 sample-nginx 이름의 docker 이미지가 생성됩니다.
```
> docker build --tag sample-nginx .		# 도커이미지 생성
> docker images					# 생성된 이미지 확인
```

## 실행
sample-nginx라는 이름으로 docker 이미지를 생성했으니 nginx를 실행해 보겠습니다.

### docker run
docker run 명령을 통해서 직접 실행할 수 있습니다.
run 명령으로 실행하게 되면 호스트가 변경되었거나 컨테이너 삭제 후 다시 실행할 경우에 실행했던 명령어를 기억하지 못해 불편할 수 있습니다.
```
> docker run -d \
     --name sample-nginx \
     -p 80:80 \
     -p 443:443 \
     --restart=always \
     --env TZ=Asia/Seoul \
     -v ~/volumes/nginx/logs:/logs \
     sample-nginx
```

### docker-compose
docker run으로 간단하게 실행해볼 수도 있지만, 실행 정보를 좀 더 구조적으로 확인하고 실행하려면 docker-compose를 이용하는 게 좋습니다.
아래 내용으로 docker-compose.yml이라는 파일을 만들면 docker-compose 명령어로 실행할 수 있습니다.
```yaml
version: '3.8'
services:
  nginx:
    build: .
    container_name: sample-nginx
    image: sample-nginx
    volumes:
      - "~/volumes/nginx/logs:/logs"
    restart: always
    ports:
      - "80:80"
      - "443:443"
    environment:
      TZ: Asia/Seoul
```
아래는 자주 사용하는 docker-compose 명령어입니다.
```
> docker-compose up 			# 실행
> docker-compose up -d 			# detach 모드 실행
> docker-compose up --no-deps	    # 연결된 다른 서비스 제외하고 시작
> docker-compose up --build		# 컨테이너를 시작하기 전에 이미지를 빌드한다.

> docker-compose stop 			# 컨테이너 정지
> docker-compose pause			# 컨테이너 일시정지
> docker-compose restart 		# 컨테이너 재시작
> docker-compose rm			# 컨테이너 삭제
> docker-compose down			# 컨테이너 제거(네트워크, 볼륨 전체)

> docker-compose ps			# 실행중인 컨테이너 확인
> docker-compose logs			# 컨테이너 로그 확인
```
`docker-compose up -d`를 입력해서 docker-compose를 실행하고 http://localhost로 접속하면 **Welcome to nginx!** 페이지를 확인할 수 있습니다.

## Logrotate 적용 확인
logrotate가 잘 적용되었는지 확인하려면 컨테이너로 접속해서 확인해 볼 수 있습니다. 아래 명령어로 실행중인 컨테이너 안을 확인할 수 있습니다.
```
> docker exec -it <CONTAINER_ID> /bin/sh
```
접속한 후에 logrotate 명령어로 이상 없이 실행가능한지 확인할 수 있습니다.
```
> logrotate -v /etc/logrotate.d/nginx
> cat /var/lib/logrotate.status
```

끝.