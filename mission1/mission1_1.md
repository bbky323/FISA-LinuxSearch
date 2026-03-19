#  ❓문제 1 - 장애 발생 후 에러 로그 추적 문제

## 📚가정 상황
- 운영 중인 웹 서비스에서 새벽에 장애가 발생했다.
- 개발팀은 “최근 수정된 로그 파일 중 에러가 집중된 파일부터 확인해 달라”고 요청했다.
## 🤔문제
- /linux_mission1/log 아래에서 다음 조건을 만족하는 로그 파일을 찾고, 해당 파일들에서 장애 원인이 될 만한 로그를 추출하시오.
    - 파일 이름이 .log 또는 .out로 끝나는 일반 파일
    - 최근 3일 이내 수정된 파일
    - 파일 크기가 0C 이상인 파일
    - 이후 찾은 파일들에서 아래 패턴이 포함된 줄만 추출하여 error_report.txt에 저장하시오.
    - ERROR / FATAL / Exception / timeout
    - 추가로, **파일별 에러 발생 건수**를 정리하여 함께 출력하시오.
    - Keyword: find, grep, awk
## 📝풀이
### 1단계. 조건에 맞는 파일 찾기

> 최근 3일 이내 수정된 `.log`, `.out` 일반 파일만 검색한다.

```bash
find "./linux_mission1/log" -type f \( -name "*.log" -o -name "*.out" \) -mtime -3 -size +0c
```

| 옵션 | 의미 |
|------|------|
| `find` | 파일/디렉토리를 검색하는 명령어 |
| `"./linux_mission1/log"` | 검색 시작 경로 (`.`은 현재 디렉토리) |
| `-type f` | 일반 파일만 검색 |
| `\(` `\)` | 조건 그룹화 |
| `-name "*.log"` | `.log` 파일 검색 |
| `-o` | OR 조건 |
| `-name "*.out"` | `.out` 파일 검색 |
| `-mtime -3` | 최근 3일 이내 수정된 파일 |
| `-size +0c` | 0바이트 초과 파일 |

---

### 2단계. 찾은 파일에서 에러 패턴 추출하기

> 검색된 각 파일에서 `ERROR`, `FATAL`, `Exception`, `timeout` 문자열이 포함된 줄만 추출한다.

```bash
find "./linux_mission1/log" -type f \( -name "*.log" -o -name "*.out" \) -mtime -3 -size +0c -exec grep -HnE 'ERROR|FATAL|Exception|timeout' {} \;
```

| 옵션 | 의미 |
|------|------|
| `-exec` | `find`가 찾은 각 파일마다 뒤 명령어 실행 |
| `grep` | 파일 내용에서 특정 패턴 검색 |
| `-H` | 파일 이름 출력 |
| `-n` | 줄 번호 출력 |
| `-E` | 확장 정규표현식 사용 |
| `'ERROR\|FATAL\|Exception\|timeout'` | 네 가지 문자열 중 하나라도 포함된 줄 검색 |
| `{}` | 현재 검색된 파일 경로가 들어가는 자리 |
| `\;` | `-exec` 종료 표시 |

---

### 3단계. 검색 결과를 파일로 저장하기

> 추출한 에러 패턴 결과를 `error_report.txt` 파일에 저장하고 내용을 확인한다.

```bash
find "./linux_mission1/log" -type f \( -name "*.log" -o -name "*.out" \) -mtime -3 -size +0c -exec grep -HnE 'ERROR|FATAL|Exception|timeout' {} \; > error_report.txt
cat error_report.txt
```

| 옵션 | 의미 |
|------|------|
| `>` | 표준출력을 파일로 저장 |
| `error_report.txt` | 검색 결과 저장 파일 |
| `cat` | 파일 내용을 그대로 출력 |

---

### 4단계. 파일별 에러 개수 집계하기

> `error_report.txt`를 읽어서 각 파일별로 몇 줄이 매칭되었는지 집계한다.

```bash
awk -F: '{count[$1]++} END {for (f in count) print count[f], f}' error_report.txt
```

| 옵션/문법 | 의미 |
|----------|------|
| `awk` | 텍스트를 필드(열) 단위로 처리하는 명령어 |
| `-F:` | 필드 구분자를 `:` 로 지정 |
| `count[$1]++` | 첫 번째 필드(파일명)를 기준으로 개수 증가 |
| `END` | 모든 줄을 다 읽은 뒤 실행 |
| `for (f in count)` | 집계된 파일명을 하나씩 순회 |
| `print count[f], f` | 파일별 개수와 파일명 출력 |


### 5단계. 에러 개수가 많은 순으로 정렬하기

> 집계한 결과를 숫자 기준 내림차순으로 정렬하여 에러가 가장 많은 파일부터 확인한다.

```bash
awk -F: '{count[$1]++} END {for (f in count) print count[f], f}' error_report.txt | sort -nr
```

| 옵션 | 의미 |
|------|------|
| `|` | 왼쪽 명령어 결과를 오른쪽 명령어 입력으로 전달 |
| `sort` | 정렬 명령어 |
| `-n` | 숫자 기준 정렬 |
| `-r` | 역순 정렬 (내림차순) |


## ✅정답 결과

<img width="1632" height="338" alt="image" src="./mission1_1.png" />
