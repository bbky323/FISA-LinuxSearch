## 🛡️상황 가정

보안팀으로부터 다음과 같은 알림을 받았다.

“어젯밤 SSH 로그인 실패 횟수가 비정상적으로 증가했습니다.”

서버 관리자는

 **어떤 IP에서 얼마나 로그인 시도를 했는지**

 **공격 의심 IP를 빠르게 식별해야 한다.**

---

## ❓문제

시스템 인증 로그 (auth.log)를 분석하여 다음을 수행하시오. 

### 요구사항

1. **SSH 로그인 실패 기록만 추출**
2. **실패 요청을 보낸 IP 주소별 시도 횟수 집계**
3. 결과를 ssh_attack_summary.txt파일에 저장

### 조건

- 다음과 같은 다양한 실패 로그를 고려해야 한다
    - Failed password
    - Invalid user
    - authentication failure
- grep + 정규표현식 + awk + 파이프라인을 활용할 것

### 파일 생성

```
nano auth.log
```

아래 데이터 붙여넣기

```jsx
Mar 19 01:23:45 server sshd[1234]: Failed password for invalid user admin from 192.168.1.10 port 22
Mar 19 01:23:50 server sshd[1235]: Failed password for root from 192.168.1.10 port 22
Mar 19 01:24:02 server sshd[1236]: Failed password for invalid user test from 10.0.0.5 port 22
Mar 19 01:24:10 server sshd[1237]: Invalid user guest from 10.0.0.5 port 22
Mar 19 01:24:25 server sshd[1238]: Failed password for root from 172.16.0.3 port 22
Mar 19 01:24:30 server sshd[1239]: authentication failure; rhost=192.168.1.10 user=root
Mar 19 01:24:45 server sshd[1240]: Failed password for invalid user oracle from 172.16.0.3 port 22
Mar 19 01:25:01 server sshd[1241]: Invalid user admin from 203.0.113.7 port 22
Mar 19 01:25:15 server sshd[1242]: Failed password for root from 203.0.113.7 port 22
Mar 19 01:25:30 server sshd[1243]: authentication failure; rhost=192.168.1.10 user=admin
Mar 19 01:25:45 server sshd[1244]: Failed password for invalid user test from 10.0.0.5 port 22
Mar 19 01:26:00 server sshd[1245]: Failed password for root from 198.51.100.9 port 22
Mar 19 01:26:10 server sshd[1246]: Invalid user user1 from 198.51.100.9 port 22
Mar 19 01:26:25 server sshd[1247]: authentication failure; rhost=198.51.100.9 user=test
Mar 19 01:26:40 server sshd[1248]: Failed password for root from 172.16.0.3 port 22
Mar 19 01:26:55 server sshd[1249]: Failed password for invalid user guest from 203.0.113.7 port 22
Mar 19 01:27:10 server sshd[1250]: Invalid user test from 10.0.0.5 port 22
Mar 19 01:27:25 server sshd[1251]: authentication failure; rhost=172.16.0.3 user=root
Mar 19 01:27:40 server sshd[1252]: Failed password for root from 192.168.1.10 port 22
Mar 19 01:27:55 server sshd[1253]: Failed password for invalid user admin from 203.0.113.7 port 22
```

---

## 💡풀이

```jsx
grep "Failed password" auth.log > temp.txt
grep "Invalid user" auth.log >> temp.txt
grep "authentication failure" auth.log >> temp.txt

grep -Eo "([0-9]{1,3}\.){3}[0-9]{1,3}" temp.txt \
| awk '{count[$1]++} END {for (ip in count) print count[ip], ip}'\
> ssh_attack_summary.txt
```

## 📊결과



