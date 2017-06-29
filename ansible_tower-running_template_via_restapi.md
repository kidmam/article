# Ansible Tower - RestAPI를 통해 Playbook 실행하는 방법
## Objective
Ansible Tower의 Rest API를 통해 Job Template을 실행하는 방법 도출
- host_key 인증을 통해서 외부의 서버에서 Job Template 실행 
## Test Environment
- ansible tower: 3.1.2
- ansible: 2.3.1
- Control Node
 - ansible-tower: 192.168.56.102
- Managed Nodes
 - rhel71: 192.168.56.101
 - rhel72: 192.168.56.110
 
## snippet of sample playbook
```
---
- name: simple api test playbook
  hosts: rhel71
  vars:
    - var1: "yongki"
    - var2: "alex"
  tasks:
    - name: touch meaningless file
      shell:
        "touch /tmp/foo; echo '$(date), {{ var1 }} {{ var2 }}' >> /tmp/foo"
    - name: print msg
      debug:
        msg: " variable is {{ var1 }} {{ var2 }} "
```

## Job template configuration
host_key를 사용해서 job template를 실행할 때는, api 호출을 시도하는 호스트가 플레이북의 "hosts: " 항목에 들어가 있어야한다.
그렇치 않을 경우에는 자신이 인증하지 않은 서버로 판단하여 해당 명령이 실행되지 않는다.
즉, 플레이북에서 "hosts: rhel71" 로 정의되었다면, 아래 call_simple_command.sh 명령은 rhel72에서만 정상적으로 작동한다.
당연한 얘기지만, 이 작업 전에는 rhel71 호스트가 먼저 inventory에 등록되어야 한다.

## test to run template using curl
이테스트를 위한 참고 문서는 아래와 같다.
- http://docs.ansible.com/ansible-tower/3.1.3/html/administration/tipsandtricks.html#launch-jobs-curl
- cat /usr/share/awx/request_tower_configuration.sh (on tower system)

### extra_var 아규먼트없이 실행 
먼저 Jobtemplate 설정 화면에서 만든 host_key를 기반으로 curl 스크립트를 작성한다.

1. REST API를 호출할 스크립트 생성
아래 스크립트는  rhel71 호스트에 생성

cat /root/call_simple_command.sh
``` sh
#!/bin/sh
curl -vvv -k --data "host_config_key=ac3ed6b919cc13b34fbf0b743d3a7efb" \
https://192.168.56.102:443/api/v1/job_templates/7/callback/
```

2. 스크립트로 템플릿 실행
해당 명령은 플레이북에 정의된 rhel71 에서 실행한다.

```
[root@rhel71 ~]# sh provison_callback.sh
* About to connect() to 192.168.56.102 port 443 (#0)
*   Trying 192.168.56.102...
* Connected to 192.168.56.102 (192.168.56.102) port 443 (#0)
* Initializing NSS with certpath: sql:/etc/pki/nssdb
* skipping SSL peer certificate verification
* SSL connection using TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
* Server certificate:
*       subject: CN=www.ansible.com,O=Ansible,L=Raleigh,ST=NC,C=US
*       start date: Apr 05 17:08:40 2017 GMT
*       expire date: Jan 18 17:08:40 2291 GMT
*       common name: www.ansible.com
*       issuer: CN=www.ansible.com,O=Ansible,L=Raleigh,ST=NC,C=US
> POST /api/v1/job_templates/7/callback/ HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 192.168.56.102
> Accept: */*
> Content-Length: 48
> Content-Type: application/x-www-form-urlencoded
>
* upload completely sent off: 48 out of 48 bytes
< HTTP/1.1 201 CREATED
< Server: nginx/1.10.2
< Date: Thu, 29 Jun 2017 02:02:26 GMT
< Transfer-Encoding: chunked
< Connection: keep-alive
< X-API-Time: 0.379s
< Allow: GET, POST, HEAD, OPTIONS
< Content-Language: en
< Vary: Accept, Accept-Language, Cookie
< Location: https://192.168.56.102/api/v1/jobs/68/
< X-API-Node: localhost
< Strict-Transport-Security: max-age=15768000
< X-Frame-Options: DENY
<
* Connection #0 to host 192.168.56.102 left intact
```
3. 실행 결과
- 원래 변수인 var1: yongki, var2: alex 가 정상적으로 출력된다.

<< no_vars_curl >> 이미지 추가 >>

### extra_var 아규먼트를 추가하여 실행
1. REST API를 호출할 스크립트 생성
아래 스크립트는  rhel71 호스트에 생성

cat /root/call_simple_command_with_extra_vars.sh
``` sh
#!/bin/sh
curl -k -f -H 'Content-Type: application/json' -XPOST  \
-d '{"host_config_key": "ac3ed6b919cc13b34fbf0b743d3a7efb", "extra_vars": "{\"var1\": \"hello\"}"}' \ 
https://192.168.56.102:443/api/v1/job_templates/7/callback/
```

2. 스크립트로 템플릿 실행
해당 명령은 플레이북에 정의된 rhel71 에서 실행한다.

```
[root@rhel71 ~]# sh call_simple_command_with_extra_vars.sh
* About to connect() to 192.168.56.102 port 443 (#0)
*   Trying 192.168.56.102...
* Connected to 192.168.56.102 (192.168.56.102) port 443 (#0)
* Initializing NSS with certpath: sql:/etc/pki/nssdb
* skipping SSL peer certificate verification
* SSL connection using TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
* Server certificate:
*       subject: CN=www.ansible.com,O=Ansible,L=Raleigh,ST=NC,C=US
*       start date: Apr 05 17:08:40 2017 GMT
*       expire date: Jan 18 17:08:40 2291 GMT
*       common name: www.ansible.com
*       issuer: CN=www.ansible.com,O=Ansible,L=Raleigh,ST=NC,C=US
> POST /api/v1/job_templates/7/callback/ HTTP/1.1
> User-Agent: curl/7.29.0
> Host: 192.168.56.102
> Accept: */*
> Content-Type: application/json
> Content-Length: 95
>
* upload completely sent off: 95 out of 95 bytes
< HTTP/1.1 201 CREATED
< Server: nginx/1.10.2
< Date: Thu, 29 Jun 2017 02:18:00 GMT
< Transfer-Encoding: chunked
< Connection: keep-alive
< X-API-Time: 0.190s
< Allow: GET, POST, HEAD, OPTIONS
< Content-Language: en
< Vary: Accept, Accept-Language, Cookie
< Location: https://192.168.56.102/api/v1/jobs/70/
< X-API-Node: localhost
< Strict-Transport-Security: max-age=15768000
< X-Frame-Options: DENY
<
* Connection #0 to host 192.168.56.102 left intact
```
3. 실행 결과
- extra_vars 로 설정한 var1: hello 로 출력되지 않고, 원래 변수인 var1: yongki 출력되었다.

<< no_vars_curl >> 이미지 추가 >>

4. 디버깅
이슈 해결을 위해 권고하는대로 Ansible Tower GUI의 Job Template에서 "Prompt on launch" 를  활성화시켰다.
하지만 아래와 같은 메시지가 발생하며 에러가 발생하였다.

```
curl: (22) The requested URL returned error: 405 METHOD NOT ALLOWED
```

그래서 다음 방법인 tower-cli 로 방법을 전환하였다.(with sobbing)

## test to run template using tower-cli
### tower-cli 설치
