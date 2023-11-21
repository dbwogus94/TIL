## 1. Nginx에서 말하는 프록싱

- 프록싱은 일반적으로 여러 서버에 로드를 분산하는 작업에 사용된다.
- 다른 웹사이트의 콘텐츠를 매끄럽게 표시하는 작업에 사용된다.
- HTTP 이외의 프로토콜을 통해 처리 요청을 에플리케이션(WAS) 서버로 전달하는데 사용된다.
- “CSR - API 서버” 구조를 사용시 Nginx를 사용하여 API 서버를 프록시하는 방법을 많이 사용한다.

## 2. 프록시 서버에 요청 전달

Nginx의 프록싱은 요청을 뒷단 서버로 보내고, 처리된 결과를 받아와 클라이언트로 응답한다.

- Nginx는 지정된 프로토콜을 사용하여 HTTP 서버(Nginx이외의 또 다른 서버)에 요청을 전달한다.
- 또한 HTTP가 아닌 서버에도 요청을 전달(프록시) 할 수 있다.
- 지원되는 프로토콜은 FastCGI, uwsgi, SCGI 및 memcached이 있다.

### 2.1 프록시 서버에 요청을 전달하는 지시문 `proxy_pass`
([Module ngx_http_proxy_module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass) 참고). 

proxy_pass 지시문은 location 블록 내부에 정의한다.

```nginx
location /some/path/ {
	# 해당 경로로 들어오는 요청은 모두 프록싱 된다.
	proxy_pass <http://www.example.com/link/>;
}
```

### 2.2 프로토콜에 따른 `**_pass` 지시문

- [fastcgi_pass](https://nginx.org/en/docs/http/ngx_http_fastcgi_module.html#fastcgi_pass) 지시문은 FastCGI 서버에 요청을 전달합니다.
- [uwsgi_pass](https://nginx.org/en/docs/http/ngx_http_uwsgi_module.html#uwsgi_pass) 지시문은 uwsgi 서버에 요청을 전달합니다.
- [scgi_pass](https://nginx.org/en/docs/http/ngx_http_scgi_module.html#scgi_pass) 지시문은 SCGI 서버에 요청을 전달합니다.
- [memcached_pass](https://nginx.org/en/docs/http/ngx_http_memcached_module.html#memcached_pass) 지시문은 memcached 서버에 요청을 전달합니다.

## 3. 요청 헤더 전달

기본적으로 Nginx는 프록시된 요청에서 두 필드 "Host" 및 "Connection"를 재정의한다.
그리고 값이 빈 문자열인 헤더 필드를 제거한다.
- "Host" 필드는 `$proxy_host` 변수로 설정된다. 
- "Connection"은 `close`로 설정된다.

### 3.1 `proxy_set_header`를 사용한 요청 헤더 수정
([Module ngx_http_proxy_module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_set_header) 참고) 

프록시를 처리하는 location 블록에 `proxy_set_heade. `을 사용하여 요청 헤더를 수정할 수 있다.  
```nginx
location /some/path/ {
	# Host 헤더를 $host변수의 값으로 수정
	proxy_set_header Host $host;
	# X-Forwarded-For와 동일하게 애플리케이션에서 
	# Client IP를 확인하기 위해 사용하는 헤더값을 말한다.
	proxy_set_header X-Real-IP $remote_addr;
	proxy_pass <http://localhost:8000>;
}
```

만약 헤더 필드의 값을 빈값으로 변경 하려면 빈 문자열(`""`)로 변경한다.  
```nginx
location /some/path/ {
  proxy_set_header Accept-Encoding "";
  proxy_pass <http://localhost:8000>;
}
```

## 4. 버퍼 구성

기본적으로 **Nginx는 프록시 서버의 응답을 버퍼링한다.**  
프록시의 응답은 내부 버퍼에 저장되며 전체 응답이 수신될 때까지 클라이언트에 전송되지 않는다.  
Nginx에서 기본적으로 버퍼링을 사용하는 것은 성능 최적화에 도움이 되기 때문이다.  

Nginx의 버퍼링을 사용하지 않으면 응답은 Nginx에서 클라이언트로 동기적으로 전달된다.  
이 때 네트워크 지연 등의 문제로 클라이언트가 느리다면 프록시 서버의 시간을 낭비할 수 있다.

반면 버퍼링을 활성화하면 Nginx는 프록시 서버가 보낸 응답이 모두 수신될 때까지 버퍼링하고 수신이 완료될 때 클라이언트에게 응답한다.  

즉, 버퍼링이 활성화 사용하는 것으로 Nginx는 프록시 서버가 응답을 빠르게 처리할 수 있도록 하고 클라이언트는 다운로드하는 데 필요한 시간 만큼 응답을 저장하여 효율적인 응답을 하게 한다.

### 4.1 기본 버퍼 수를 조절하는 설정

```nginx
location /some/path/ {
	# 기본 버퍼 수를 늘리고, 응답의 첫 번째에 대한 버퍼 크기를 기본값보다 작게 만드는 설정
    proxy_buffers 16 4k;  
    proxy_buffer_size 2k;
    proxy_pass <http://localhost:8000>;
}
```

### 4.2 버퍼링을 비활성화 하는 설정

버퍼링을 비활성화 하는경우 응답은 프록시 서버에서 응답을 수신하는 동안 동기적으로 전송된다.
이러한 설정은 가능한 데이터가 큰 경우에는 맞지 않는다.  
반면 **응답 수신을 가능한 빨리 해야하는 대화형 클라이언트(채팅)에게는 바람직 할 수 있다.**

```nginx
location /some/path/ {
	# 버퍼링 모드 비활성화
    proxy_buffering off;
    proxy_pass <http://localhost:8000>;
}
```

위 경우 Nginx는 [proxy_buffer_size](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_buffer_size) 로 구성된 버퍼만 사용 하여 응답의 현재 부분을 저장한다.

또한 리버스 프록시의 일반적인 용도는 로드 밸런싱을 제공하는 것이며, Nginx에서는 이러한 방법을 가이드 하는 "[소프트웨어 로드 밸런서를 선택해야 하는 5가지 이유](https://www.nginx.com/resources/library/five-reasons-choose-software-load-balancer/)"라는 가이드를 제공한다.

## 5. 발신 IP 주소 선택

프록싱 되는 서버에 여러 네트워크 인터페이스가 있는 경우   
다음 프록시 서버나 업스트림에 연결하기 위해 특정 IP 주소를 부여해야 할 수도 있다.  
(업스트림은 프록시 서버 뒷단에 실제 요청을 처리하는 서버를 말한다.)  

이 경우는 프록시 서버가 특정 IP 네트워크 또는 IP 주소범위의 연결을 허용하도록  
구성된 경우 유용하게 사용될 수 있다.
### 5.1. proxy_bind 지시문을 사용하여 IP 주소 지정
([Module ngx_http_proxy_module](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_bind) 참고)

`proxy_bind` 지시문을 사용하여 필요한 **네트워크 인터페이스의 IP 주소를 지정(NIC 카드)** 한다.  

```nginx
location /app1/ {
    proxy_bind 127.0.0.1; # 바인드할 IP 주소범위
    proxy_pass <http://example.com/app1/>;
}

location /app2/ {
    proxy_bind 127.0.0.2; # 바인드할 IP 주소범위
    proxy_pass <http://example.com/app2/>;
}
```

### 5.2 변수를 사용하여 IP 주소 지정

[$server_addr](https://nginx.org/en/docs/http/ngx_http_core_module.html#var_server_addr)변수에는 요청을 수락한 네트워크 인터페이스의 IP 주소가 전달된다.

```nginx
location /app3/ {
    proxy_bind $server_addr;
    proxy_pass <http://example.com/app3/>;
}
```


## 참고
- [NGINX Reverse Proxy](https://docs.nginx.com/nginx/admin-guide/web-server/reverse-proxy/)