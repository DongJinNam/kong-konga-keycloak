# Records

### 초기 설정

```sh
$ docker-compose up
```


### 기존 정보 제거

route

```sh
curl -s -X DELETE http://localhost:8001/services/mock-service/routes/{route_id}
```

service

```sh
curl -s -X DELETE http://localhost:8001/services/{service_id}
```

plugin

```sh
curl -s -X DELETE http://localhost:8001/plugins/{plugin_id}
```


### mockbin 활용 서비스 생성

Request
```sh
curl -s -X POST http://localhost:8001/services \
    -d name=mock-service \
    -d url=http://mockbin.org/request \
    | python -mjson.tool
```

Response
```
{
    "host": "mockbin.org",
    "created_at": 1637900381,
    "connect_timeout": 60000,
    "id": "2e9a9d76-33f8-424a-9794-a56ec4ce6606",
    "protocol": "http",
    "name": "mock-service",
    "read_timeout": 60000,
    "port": 80,
    "path": "/request",
    "updated_at": 1637900381,
    "retries": 5,
    "write_timeout": 60000,
    "tags": null,
    "client_certificate": null
}
```

### 생성된 service id 와 연결된 route 생성

Request
* services/{id}/routes : id 는 service 등록 시 전달받은 response 내 id

```sh
curl -s -X POST http://localhost:8001/services/mock-service/routes -d "paths[]=/mock" \
    | python -mjson.tool
```

Response
```
{
    "id": "c52bbc65-324f-4697-a6da-c27cac15c216",
    "path_handling": "v0",
    "paths": [
        "/mock"
    ],
    "destinations": null,
    "headers": null,
    "protocols": [
        "http",
        "https"
    ],
    "methods": null,
    "snis": null,
    "service": {
        "id": "2e9a9d76-33f8-424a-9794-a56ec4ce6606"
    },
    "name": null,
    "strip_path": true,
    "preserve_host": false,
    "regex_priority": 0,
    "updated_at": 1637900573,
    "sources": null,
    "hosts": null,
    "https_redirect_status_code": 426,
    "tags": null,
    "created_at": 1637900573
}
```

### API Call

Request
```sh
$ curl -s http://localhost:8000/mock
```

Response
```sh
{
  "startedDateTime": "2021-11-26T04:43:46.908Z",
  "clientIPAddress": "172.18.0.1",
  "method": "GET",
  "url": "http://localhost/request",
  "httpVersion": "HTTP/1.1",
  "cookies": {},
  "headers": {
    "host": "mockbin.org",
    "connection": "close",
    "accept-encoding": "gzip",
    "x-forwarded-for": "172.18.0.1,118.235.11.218, 162.158.5.191",
    "cf-ray": "6b4075303a11ae67-KIX",
    "x-forwarded-proto": "http",
    "cf-visitor": "{\"scheme\":\"http\"}",
    "x-forwarded-host": "localhost",
    "x-forwarded-port": "80",
    "user-agent": "curl/7.71.1",
    "accept": "*/*",
    "cf-connecting-ip": "118.235.11.218",
    "cdn-loop": "cloudflare",
    "x-request-id": "52fb909f-d7b2-4ae6-8071-7a89b96132eb",
    "via": "1.1 vegur",
    "connect-time": "0",
    "x-request-start": "1637901826903",
    "total-route-time": "0"
  },
  "queryString": {},
  "postData": {
    "mimeType": "application/octet-stream",
    "text": "",
    "params": []
  },
  "headersSize": 530,
  "bodySize": 0
}%    
```


### Keycloak 설정 후 kong oidc plugin 연결

Request
* 아래 HOST_IP 는 로컬 호스트가 아닌 로컬pc가 연결된 ip주소를 의미함
* bearer_only 는 리다이렉트 없이 토큰 검증할 것인지 체크하는 변수

```sh
$ HOST_IP={local ip}
$ CLIENT_SECRET={kong 생성하면서 만든 client secret}
$ REALM={keycloak 에서 생성한 realm name}
...
curl -s -X POST http://localhost:8001/plugins \
  -d name=oidc \
  -d config.client_id=kong \
  -d config.client_secret=${CLIENT_SECRET} \
  -d config.bearer_only=yes \
  -d config.realm=${REALM} \
  -d config.introspection_endpoint=http://${HOST_IP}:8180/auth/realms/${REALM}/protocol/openid-connect/token/introspect \
  -d config.discovery=http://${HOST_IP}:8180/auth/realms/${REALM}/.well-known/openid-configuration \
  | python -mjson.tool
{
    "created_at": 1638251930,
    "config": {
        "response_type": "code",
        "introspection_endpoint": "http://192.168.0.7:8180/auth/realms/experimental/protocol/openid-connect/token/introspect",
        "filters": null,
        "bearer_only": "yes",
        "ssl_verify": "no",
        "session_secret": null,
        "introspection_endpoint_auth_method": null,
        "realm": "experimental",
        "redirect_after_logout_uri": "/",
        "scope": "openid",
        "token_endpoint_auth_method": "client_secret_post",
        "logout_path": "/logout",
        "client_id": "kong",
        "client_secret": "ff22e0b4-a8d4-4fad-b455-ad051992bf56",
        "discovery": "http://192.168.0.7:8180/auth/realms/experimental/.well-known/openid-configuration",
        "recovery_page_path": null,
        "redirect_uri_path": null
    },
    "id": "3a9d539c-ac65-44cb-a74e-4610b7f2c022",
    "service": null,
    "enabled": true,
    "protocols": [
        "grpc",
        "grpcs",
        "http",
        "https"
    ],
    "name": "oidc",
    "consumer": null,
    "route": null,
    "tags": null
}
```


### Final Test

**token 없이 api call**

request
```sh
$ curl "http://${HOST_IP}:8000/mock" \
-H "Accept: application/json" -I
```

response
* 401 error

**get token**

request
* username : keycloak 에 생성된 username
* password : password
* client_id : 로그인할 client

request
```sh
RAWTKN=$(curl -s -X POST \
        -H "Content-Type: application/x-www-form-urlencoded" \
        -d "username=demouser" \
        -d "password=demouser" \
        -d "grant_type=password" \
        -d "client_id=myapp" \
        http://${HOST_IP}:8180/auth/realms/${REALM}/protocol/openid-connect/token \
        |jq . )
```

response
```sh
{
    "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJ2WEwzd0lyVlpMZ0hCSTQzUS12UnJCMkdraVZaYlNSeUZ5MFd1TGFkemNnIn0.eyJleHAiOjE2Mzc5MDM1OTIsImlhdCI6MTYzNzkwMzI5MiwianRpIjoiYzAwMDlhNGUtNWE0OC00OTkxLWJmMWUtNjNmMjNiNzQwYTE1IiwiaXNzIjoiaHR0cDovLzE3Mi4yMC4xMC4yOjgxODAvYXV0aC9yZWFsbXMvZXhwZXJpbWVudGFsIiwiYXVkIjoiYWNjb3VudCIsInN1YiI6IjNiNTJiMjkwLTVmMDgtNGQyMi1iNzRjLTM2NWYxZWViZjQwMyIsInR5cCI6IkJlYXJlciIsImF6cCI6Im15YXBwIiwic2Vzc2lvbl9zdGF0ZSI6Ijk4OWY3ZThkLTcxM2YtNDkzNC05NDQ2LWMwNDIyYTdlNGNlZSIsImFjciI6IjEiLCJyZWFsbV9hY2Nlc3MiOnsicm9sZXMiOlsib2ZmbGluZV9hY2Nlc3MiLCJ1bWFfYXV0aG9yaXphdGlvbiJdfSwicmVzb3VyY2VfYWNjZXNzIjp7ImFjY291bnQiOnsicm9sZXMiOlsibWFuYWdlLWFjY291bnQiLCJtYW5hZ2UtYWNjb3VudC1saW5rcyIsInZpZXctcHJvZmlsZSJdfX0sInNjb3BlIjoiZW1haWwgcHJvZmlsZSIsImVtYWlsX3ZlcmlmaWVkIjp0cnVlLCJuYW1lIjoidGVzdGVyIiwicHJlZmVycmVkX3VzZXJuYW1lIjoiZGVtb3VzZXIiLCJnaXZlbl9uYW1lIjoidGVzdGVyIiwiZW1haWwiOiJ0ZXN0QHRlc3QuY29tIn0.CcbSRY7oZGxJNtmTzLun0gBz1YHV4Xr6SVmqtR2jJ71j7Z3oAaiQQ7_iEr4kj2SffXxbjY1pgDyZ9Pv99ZEe0tF8Tl08-iiY4WTpKIMu5EU_ZvtskkwlIWo3iWTDWKoRJmmbj1bEuuGKBWYSEetjbAPzCXSJoHPA9mShNMxmxCcvOrdnP-izPKWXsZ4a7r5SEG7ByYqa2s_y8pGWygpkf_RG3kaFdj03mLQa4aWWw3BdEEjtSXkZa6O7wrL1mRWKEEmfaLgpnD94VoSZHefmfV40PI9-5TgN-5axhTqpdzCzR0DKqEnoueNr6lad6wGUBH-l6pEUtBcAyH_qMIcZDA",
    "expires_in": 300,
    "refresh_expires_in": 1800,
    "refresh_token": "eyJhbGciOiJIUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICIwNjc5YmE0ZS0wN2RmLTQ2ZmItOGIwYi1jYzI2MWNmY2VkMjUifQ.eyJleHAiOjE2Mzc5MDUwOTIsImlhdCI6MTYzNzkwMzI5MiwianRpIjoiYzI5ZTk0ZTItZGMxZC00MDg4LThjYmMtZmU2NmJkM2VmNTk5IiwiaXNzIjoiaHR0cDovLzE3Mi4yMC4xMC4yOjgxODAvYXV0aC9yZWFsbXMvZXhwZXJpbWVudGFsIiwiYXVkIjoiaHR0cDovLzE3Mi4yMC4xMC4yOjgxODAvYXV0aC9yZWFsbXMvZXhwZXJpbWVudGFsIiwic3ViIjoiM2I1MmIyOTAtNWYwOC00ZDIyLWI3NGMtMzY1ZjFlZWJmNDAzIiwidHlwIjoiUmVmcmVzaCIsImF6cCI6Im15YXBwIiwic2Vzc2lvbl9zdGF0ZSI6Ijk4OWY3ZThkLTcxM2YtNDkzNC05NDQ2LWMwNDIyYTdlNGNlZSIsInNjb3BlIjoiZW1haWwgcHJvZmlsZSJ9.kO2MGF-W83ALsXp8e56AZcTVoZZNrk7Xnav30QbUeZA",
    "token_type": "Bearer",
    "not-before-policy": 0,
    "session_state": "989f7e8d-713f-4934-9446-c0422a7e4cee",
    "scope": "email profile"
}
```

**retry request with token**

request
```sh
$ TKN={위 response 의 access_token}
$ curl "http://${HOST_IP}:8000/mock" \
-H "Accept: application/json" \
-H "Authorization: Bearer $TKN"
```


response
```sh
{
  "startedDateTime": "2021-11-26T05:10:48.130Z",
  "clientIPAddress": "172.18.0.1",
  "method": "GET",
  "url": "http://172.20.10.2/request",
  "httpVersion": "HTTP/1.1",
  "cookies": {},
  "headers": {
    "host": "mockbin.org",
    "connection": "close",
    "accept-encoding": "gzip",
    "x-forwarded-for": "172.18.0.1,118.235.11.218, 172.70.49.197",
    "cf-ray": "6b409cc4b8260ace-KIX",
    "x-forwarded-proto": "http",
    "cf-visitor": "{\"scheme\":\"http\"}",
    "x-forwarded-host": "172.20.10.2",
    "x-forwarded-port": "80",
    "user-agent": "curl/7.71.1",
    "accept": "application/json",
    "authorization": "Bearer eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJ2WEwzd0lyVlpMZ0hCSTQzUS12UnJCMkdraVZaYlNSeUZ5MFd1TGFkemNnIn0.eyJleHAiOjE2Mzc5MDM1OTIsImlhdCI6MTYzNzkwMzI5MiwianRpIjoiYzAwMDlhNGUtNWE0OC00OTkxLWJmMWUtNjNmMjNiNzQwYTE1IiwiaXNzIjoiaHR0cDovLzE3Mi4yMC4xMC4yOjgxODAvYXV0aC9yZWFsbXMvZXhwZXJpbWVudGFsIiwiYXVkIjoiYWNjb3VudCIsInN1YiI6IjNiNTJiMjkwLTVmMDgtNGQyMi1iNzRjLTM2NWYxZWViZjQwMyIsInR5cCI6IkJlYXJlciIsImF6cCI6Im15YXBwIiwic2Vzc2lvbl9zdGF0ZSI6Ijk4OWY3ZThkLTcxM2YtNDkzNC05NDQ2LWMwNDIyYTdlNGNlZSIsImFjciI6IjEiLCJyZWFsbV9hY2Nlc3MiOnsicm9sZXMiOlsib2ZmbGluZV9hY2Nlc3MiLCJ1bWFfYXV0aG9yaXphdGlvbiJdfSwicmVzb3VyY2VfYWNjZXNzIjp7ImFjY291bnQiOnsicm9sZXMiOlsibWFuYWdlLWFjY291bnQiLCJtYW5hZ2UtYWNjb3VudC1saW5rcyIsInZpZXctcHJvZmlsZSJdfX0sInNjb3BlIjoiZW1haWwgcHJvZmlsZSIsImVtYWlsX3ZlcmlmaWVkIjp0cnVlLCJuYW1lIjoidGVzdGVyIiwicHJlZmVycmVkX3VzZXJuYW1lIjoiZGVtb3VzZXIiLCJnaXZlbl9uYW1lIjoidGVzdGVyIiwiZW1haWwiOiJ0ZXN0QHRlc3QuY29tIn0.CcbSRY7oZGxJNtmTzLun0gBz1YHV4Xr6SVmqtR2jJ71j7Z3oAaiQQ7_iEr4kj2SffXxbjY1pgDyZ9Pv99ZEe0tF8Tl08-iiY4WTpKIMu5EU_ZvtskkwlIWo3iWTDWKoRJmmbj1bEuuGKBWYSEetjbAPzCXSJoHPA9mShNMxmxCcvOrdnP-izPKWXsZ4a7r5SEG7ByYqa2s_y8pGWygpkf_RG3kaFdj03mLQa4aWWw3BdEEjtSXkZa6O7wrL1mRWKEEmfaLgpnD94VoSZHefmfV40PI9-5TgN-5axhTqpdzCzR0DKqEnoueNr6lad6wGUBH-l6pEUtBcAyH_qMIcZDA",
    "x-userinfo": "eyJhenAiOiJteWFwcCIsImlhdCI6MTYzNzkwMzI5MiwiaXNzIjoiaHR0cDpcL1wvMTcyLjIwLjEwLjI6ODE4MFwvYXV0aFwvcmVhbG1zXC9leHBlcmltZW50YWwiLCJlbWFpbCI6InRlc3RAdGVzdC5jb20iLCJnaXZlbl9uYW1lIjoidGVzdGVyIiwic3ViIjoiM2I1MmIyOTAtNWYwOC00ZDIyLWI3NGMtMzY1ZjFlZWJmNDAzIiwiaWQiOiIzYjUyYjI5MC01ZjA4LTRkMjItYjc0Yy0zNjVmMWVlYmY0MDMiLCJhY3RpdmUiOnRydWUsInVzZXJuYW1lIjoiZGVtb3VzZXIiLCJleHAiOjE2Mzc5MDM1OTIsInNjb3BlIjoiZW1haWwgcHJvZmlsZSIsImF1ZCI6ImFjY291bnQiLCJzZXNzaW9uX3N0YXRlIjoiOTg5ZjdlOGQtNzEzZi00OTM0LTk0NDYtYzA0MjJhN2U0Y2VlIiwicmVzb3VyY2VfYWNjZXNzIjp7ImFjY291bnQiOnsicm9sZXMiOlsibWFuYWdlLWFjY291bnQiLCJtYW5hZ2UtYWNjb3VudC1saW5rcyIsInZpZXctcHJvZmlsZSJdfX0sIm5hbWUiOiJ0ZXN0ZXIiLCJjbGllbnRfaWQiOiJteWFwcCIsImp0aSI6ImMwMDA5YTRlLTVhNDgtNDk5MS1iZjFlLTYzZjIzYjc0MGExNSIsImFjciI6IjEiLCJlbWFpbF92ZXJpZmllZCI6dHJ1ZSwicmVhbG1fYWNjZXNzIjp7InJvbGVzIjpbIm9mZmxpbmVfYWNjZXNzIiwidW1hX2F1dGhvcml6YXRpb24iXX0sInByZWZlcnJlZF91c2VybmFtZSI6ImRlbW91c2VyIiwidHlwIjoiQmVhcmVyIn0=",
    "cf-connecting-ip": "118.235.11.218",
    "cdn-loop": "cloudflare",
    "x-request-id": "e38c817c-c9fb-4260-a929-772630f489b0",
    "via": "1.1 vegur",
    "connect-time": "0",
    "x-request-start": "1637903448128",
    "total-route-time": "0"
  },
  "queryString": {},
  "postData": {
    "mimeType": "application/octet-stream",
    "text": "",
    "params": []
  },
  "headersSize": 2761,
  "bodySize": 0
}%  
```
