## 1. 웹 서버 설정을 위해 `http` 컨텍스트 정의

```nginx
http {
	# 웹 서버 공통 환경 설정 지시문 정의
}
```

## 2. 가상 호스트 서버 설정을 위한 `server` 블록 정의

가상 호스트 서버 이하 "가상 서버"
```nginx
http {
	server {
		
	}
}
```

## 3. `server` 블록안에 지시문 정의

```nginx
http {
	server {
		# 3.1. listen 지시문
		listen 127.0.0.1:8080;
		# 3.2. server_name 지시문
		server_name example.org www.example.org
	}	
}
```

#### 3.1. `listen` 지시문 사용 방법

`listen` 지시문에는 가상 서버가 사용할 ip와 port를 정의한다.

```yaml
# 1) ip를 생략하면 모든 ip에 대해 리슨한다.
listen 80;

# 2) localhost에 8080포트에 리슨한다.
listen 127.0.0.1:8080;

# 3) 포트를 생략하면 기본 포트(80)에 대해 리슨한다.
listen 127.0.0.1;
```

#### 3.2. `server_name` 지시문 사용 방법

`server_name` 지시문에는 서버를 구분하는 이름(도메인)을 정의한다.

```yaml
# 1) 가상 서버의 이름을 www.example.org 로 지정
server_name www.example.org;

# 2) default_server를 사용하여 기본 가상 서버로 정의
server_name default_server;

# 3) default_server는 listen에 붙일 수도 있다.
listen [::]:80 default_server;
server_name _;
```

- `server_name`은 요청 IP 주소 및 Port와 일치하는 서버가 다수인 경우 사용한다.
- `server_name`의 값으로는 정확한 이름, 정규식, 와일드 카드가 들어갈수 있다.
- Nginx에서 사용하는 정규식 문법은 Perl문법을 사용한다. ( `~`를 사용한다. ex) a포함 → `~/a/`)
- 또한 여러 이름이 `Host` 헤더와 일치하는 경우 아래와 같은 순서로 이름을 검색한다.  
  (가장 처음 일치하는 항목을 사용한다.)
    1. 정확한 이름
    2. 별표로 시작하는 가장 긴 와일드카드(예: `*.example.org` )
    3. 다음과 같이 별표로 끝나는 가장 긴 와일드카드 `mail.*`
    4. 첫 번째 일치하는 정규식(구성 파일에 나타나는 순서대로)
- `server_name`에 `default_server`를 사용하면 정체를 알 수 없는 요청이 들어 왔을때 기본적으로 처리하는 가상서버 역할을 한다(switch문의 default와 비슷한 역할을 한다.)
- `default_server`는 주로 잘못된 요청을 기본 루트로 돌리는데 사용된다.

## 4. 웹 서버의 동작설정을 위해 `location` 블록 정의

Nginx는 트래픽을 다른 프록시로 보내거나 요청 URI를 기반으로 정적파일을 제공할 수 있다.  
그리고 그러한 동작을 `location` 블록을 사용하여 정의한다.

아래는 설정은 일부 요청은 프록시 서버로 보내고, 또 다른 요청은 다른 프록시로 보내고, images 와 html 파일은 로컬 파일 시스템에 있는 정적파일을 전달하도록 만든 가상 서버이다.

```nginx
http {
	server {
		listen 127.0.0.1:80;
		server_name example.org www.example.org
	}	
	# 첫 번째 path에 images가 있는 경우 처리
	location /images/ {
	    root /var/static/images; # root는 로컬 파일 시스템을 가리킨다.
	}

	# .html 확장자를 가진 경우 처리
	location ~ \\.html? {
		root /var/static/html/;
	}
	
	# 기본 루트는 www.example1.com로 넘긴다.
	location / {
	    proxy_pass <http://www.example1.com>;
	}

	# /posts 요청은 www.example2.com로 넘긴다.
	location /posts {
	    proxy_pass <http://www.example2.com>;
	}
}
```

### 4.1. [root](https://nginx.org/en/docs/http/ngx_http_core_module.html#root)` 지시문

`location` 블록의 `root` 지시문은 파일 시스템의 정적 파일의 경로를 의미한다.  

예시) GET example.png 요청
- 요청: `GET /images/example.png`
- 응답: `/var/static/images/example.png`

### 4.2. [proxy_pass](https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass) 지시문

`location` 블록의 `proxy_pass` 지시문의 값에 정의된 URL로 요청을 전달한다.  
전달된 요청이 처리되고 응답되면 Nginx는 결과를 받아서 클라이언트에게 응답된다.  

### 4.3. location 우선 순위

1. 가장 먼저 URI의 접두사 문자열이 있는 위치와 비교한다.
	접두사 문자열 중에 가장 구체적으로 일치(길면서 완벽히 일치)하는 문자열을 선택한다.

2. 없다면 정규식으로 위치를 검색한다.
    단, `^~`수정자가 사용 되지 않았다면 정규표현식에 더 높은 우선 순위가 부여 된다.  


**요청을 처리하는 location 블록을 찾는 순서**  
1. `location`의 모든 접두사 문자열에 대해 URI를 테스트한다.
2. `=` 기호가 있는 경우는 가장 앞에 사용하는 문자열에 일치하는 항목이 발견되면 검색을 중지한다.   
   그리고 해당 `location`을 사용한다. ([= 기호는 주로 루트 / 에 사용한다.]([https://www.notion.so/Nginx-14c07a8c521b4e15b2a962dbe7fd2ff5?pvs=21](https://www.notion.so/Nginx-14c07a8c521b4e15b2a962dbe7fd2ff5?pvs=21)) )
3. `^~` 표현식을 사용하면, 가장 길게 일치하는 접두사 문자열이 있어도 검사되지 않는다.
4. 가장 길게 일치하는 접두사 문자열을 저장한다.
5. 정규식에 대해 URI를 테스트한다.
6. 가장 처음 정규식에 일치하는 문자열을 찾으면 처리를 중지하고 해당 `location`을 사용한다.
7. 정규식이 일치하지 않으면, (4.)에서 저장된 접두사 문자열에 해당하는 `location`를 사용한다.

**Q) location 블록에 `=` 기호가 루트에 사용되면?**

```nginx
location = / {
    #...
}
```

A) `=` 기호의 일반적으로 `/`(루트)에 사용한다.
그리고 웹서비스에서 `/`는 대표적인 가장 빈번하게 호출되는 URI 이다.  
Nginx는 이렇게 빈번하게 사용되는 URI에 처리 속도를 빠르게 하기 위해 `=` 기호를 지원한다.  

위에 설명한 "요청을 처리하는 location 블록을 찾는 순서"에서 처럼 location을 찾는 과정에는 7단계가 있다.  
이중 `=` 기호를 처리하는 로직은 2번째에 위치한다.

그리고 2번째 로직에서 일치하는 URI를 찾으면 더 이상의 location을 찾는 탐색을 하지 않고, 일치하는 location을 사용하여 응답한다.
**즉, `=`을 사용하면 7단계의 location을 찾는 과정 중 5단계를 스킵하기 때문에 빠르게 처리되는 것이다..**

## 5. 변수 사용 하기

설정 파일에는 변수를 사용하여, 변수에 따라 동적으로 처리할 수 있다.

- 변수는 런타임에 계산되고 지시문에 대한 매개변수로 사용되는 값이다.
- 변수는 시작 부분에 `$`를 붙여 사용한다.
- 변수는 현재 처리 중인 요청의 속성과 같은 Nginx의 상태를 기반으로 정보를 정의한다.
- nginx에는 [핵심 HTTP 변수](https://nginx.org/en/docs/http/ngx_http_core_module.html#variables)와 같은 사전 정의된 변수가 많이 있다.

### 5.1. 핵심 HTTP 변수
참고 : )

- `$arg_{Parameter}`
	여기서 Parameter는 요청 URI의 파라미터(query string)의 key 값이다.
    ex) [http://www.example1.com?**mode**=course](http://www.example1.com?**mode**=course) 로 요청이 들어오면?
    `$arg_mode`를 통해 `course`를 가져올 수 있다.
   
- `$host`
	현재 요청의 호스트명, 호스트명은 일반적으로 머신이 위치하는 IP나 도메인이다.

- `$uri`
	현재 요청의 URI. (단, 호스트명과 파라미터는 제외 된다.)

- `$document_uri` 
    `$uri`과 동일하다.

- `$args `
    URL의 질의 문자열을 가져온다.

- `$binary_remote_addr`
    바이너리 형식의 클라이언트 주소

- `$body_bytes_sent`
    전송된 바디의 바이트 수

- `$content_length`
    HTTP 요청헤더의 Content-Lenght와 동일하다.

- `$content_type`
    HTTP 요청헤더의 Content-Type와 동일하다.

- `$document_root`
    현재 요청의 document root값, root 지시어와 동일

- `$http_HEADER`  
    HTTP 헤더의 값을 소문자로 대시(-)를 밑줄()로 변환한 값.

## 6. 특정 상태 코드 반환

특정 오류 코드 또는 리다이렉션 코드가 포함된 응답을 즉시 반환하는 경우에 사용한다.

### 6.1. 404를 반환하는 경우

```nginx
location /wrong/url {
  return 404;
}
```

### 6.2. 리다이렉션

```Nginx
location /permanently/moved/url {
  return 301 <http://www.example.com/moved/here>;
}
```

## 7. 요청에서 URI를 [rewrite](<https://nginx.org/en/docs/http/ngx_http_rewrite_module.html#rewrite>) 재작성, 재정의

`rewrite` 지시문을 사용하여 요청 처리 중에 요청 URI를 여러 번 수정할 수 있다.

### 7.1. `rewrite`에는 필수 매개변수와 하나의 선택적 매개변수가 있다.

- (필수) 첫 번째 매개변수는 요청 URI와 일치하는 정규식이다.
- (필수) 두 번째 매개변수는 일치하는 URI를 대체할 URI이다.
- (선택) 세 번째 매개변수는 추가 `rewrite` 지시문 처리를 중단 하거나 리다이렉션(301, 302)을 보낼 수 있다.

### 7.2. `rewrite` 예시

예시 1) 
```nginx
location /users/ {
  rewrite ^/users/(.*)$ /show?user=$1 break;
}
```

예시 2) rewrite + return 활용
```nginx
server {
	# 1) 재작성 1.  
	rewrite ^(/download/.*)/media/(\\w+)\\.?.*$ $1/mp3/$2.mp3 last;
	# 2) 재작성 2. 
	rewrite ^(/download/.*)/audio/(\\w+)\\.?.*$ $1/mp3/$2.ra  last;
	# 3) 리턴
	return 403;
}
```

- 1) 재작성 1.  
    `/download/some/media/file` URI에 대한 요청이 있으면  
    `/download/some/mp3/file.mp3`로 재작성 된다.
    그리고 **last**에 의해 다음 재작성은 스킵된다.

- 2) 재작성 2. 
    `/download/some/audio/file` URI에 대한 요청이 있으면  
    `/download/some/mp3/file.ra`으로 재작성 된다.
    그리고 **last**에 의해 다음 재작성은 스킵된다.

- 3. 리턴
    위의 선언된 2개의 rewrite에 일치하는 URI가 아니라면 403을 리턴한다.
    ⇒ 이 방법은 `if ~ else if ~ else`문법과 비슷한 동작을 한다.

## 8. HTTP 응답 재작성 `sub_filter`
([Module ngx_http_sub_module](https://nginx.org/en/docs/http/ngx_http_sub_module.html#sub_filter) 참고)

## 9. 오류 처리 지시문 `error_page`
([Module ngx_http_core_module](https://nginx.org/en/docs/http/ngx_http_core_module.html#error_page) 참고)

Nginx는 `error_page` 지시문을 사용하여 오류 코드와 사용자 지정 페이지를 응답 할 수 있다.
또한 응답에서 다른 오류코드로 대체하거나 브라우처를 다른 URI로 리다이렉션 하도록 할 수 있다.

### 9.1 `error_page` 사용 예시

예시 1) error_page 지시문을 사용하여 오류코드와 함께 반환할 페이지를 지정하는 설정
```nginx
error_page 404 /404.html;
```
위 지시문을 사용한다고 오류가 즉시 반환되는 것은 아니다. (`return` 지시문에서 수행된다.)
위 지시문은 단순히 오류가 발생할 때 처리하는 방법을 설정하는 것이다. (오류는 내부로직에서 발생할 수 있다.)  

예시2) Nginx가 페이지를 찾을 수 없을때 404 응답을 301 코드로 대체하고 리다이렉션 하게 하는 설정
```nginx
location /old/path.html {
	error_page 404 =301 http:/example.com/new/path.html;
	# 301코드는 영구적으로 이동한 것을 브라우저에게 알려 리다이렉션을 시키는 코드이다.
}
```

예시 3) 요청 파일을 찾지 못하는 경우 백엔드로 요청을 전달하는 설정
```nginx
server {
  ...
	location /images/ {
		# 1. 요청에 대한 정적 파일 루트 설정
		root /data/www;
		
		# 2. 파일 존재와 관련된 오류 로깅 사용 안 함
		open_file_cache_errors off;
		
		# 3. 파일을 찾을 수 없는 경우 내부 리다이렉션
		error_page 404 = /fetch$uri;
	}
	
	# 4. 내부 리다이렉션될 서버 설정
	location /fetch/ {
		proxy_pass <http://backend/>;
	}
}
```

1. **/images/some/file** 요청이 들어오면? 
	`location /images/` 블록에서 해당 요청을 잡는다.

3. /images/some/file에 해당하는 정적 파일이 없다면?
	`error_page`에 의해 요청이 **/fetch/images/some/file**로 대체된다.

3. 대체된 요청 URI로 다시 처리할 **location** 블록을 찾는다.
    `location /fetch/` 블록에서 해당 요청을 잡는다.
    그리고 `proxy_pass`에 의해 요청이 `http://backend/`로 넘어간다.

## 참고
- [Configuring NGINX and NGINX Plus as a Web Server](https://docs.nginx.com/nginx/admin-guide/web-server/web-server/) 
