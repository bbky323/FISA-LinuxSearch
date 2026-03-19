🛡️ SSH 공격 로그 분석 (Linux 로그 파이프라인 실습)
📌 프로젝트 개요

보안팀으로부터 SSH 로그인 실패 횟수 급증 알림을 받고,
서버 관리자가 로그를 분석하여 공격 의심 IP를 식별하는 실습 프로젝트입니다.

🧩 상황 가정

“어젯밤 SSH 로그인 실패 횟수가 비정상적으로 증가했습니다.”

서버 관리자는 다음을 빠르게 파악해야 한다.

어떤 IP에서 로그인 시도가 발생했는가

IP별 시도 횟수는 얼마인가

공격 의심 IP는 무엇인가

❓ 문제 정의

시스템 인증 로그(auth.log)를 분석하여 다음을 수행한다.

🎯 요구사항

SSH 로그인 실패 기록만 추출

실패 요청을 보낸 IP 주소별 시도 횟수 집계

결과를 ssh_attack_summary.txt 파일로 저장

⚙️ 사용 기술
기술	설명
grep	로그 필터링
정규표현식	IP 주소 추출
awk	데이터 집계
파이프라인	명령어 연결
📄 테스트 데이터 생성
nano auth.log

아래 로그 데이터를 입력한다.

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
🚀 문제 해결 과정
1️⃣ SSH 실패 로그 추출
grep "Failed password" auth.log > temp.txt
grep "Invalid user" auth.log >> temp.txt
grep "authentication failure" auth.log >> temp.txt

👉 다양한 실패 유형을 각각 추출하여 하나의 파일로 통합

2️⃣ IP 주소 추출 및 집계
grep -Eo "([0-9]{1,3}\.){3}[0-9]{1,3}" temp.txt \
| awk '{count[$1]++} END {for (ip in count) print count[ip], ip}'
✔ 처리 과정

grep -Eo → IP 주소만 추출

awk → IP별 등장 횟수 카운트

3️⃣ 결과 파일 저장
grep -Eo "([0-9]{1,3}\.){3}[0-9]{1,3}" temp.txt \
| awk '{count[$1]++} END {for (ip in count) print count[ip], ip}' \
> ssh_attack_summary.txt
📊 실행 결과 예시
6 192.168.1.10
4 10.0.0.5
4 172.16.0.3
4 203.0.113.7
3 198.51.100.9