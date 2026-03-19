## 가정 상황
- 당신은 DevOps 엔지니어입니다. 새로운 서비스 배포 직전, 개발 환경(dev)과 운영 환경(prod)의 설정 파일 불일치로 인한 장애를 막기 위해 검수 스크립트를 작성해야 합니다.
- 
## 문제
- 4-1.
   config/ 폴더 내에서 application-으로 시작하는 .yaml 파일을 찾고, application-prod.yaml의 전체 구조를 Properties 형식으로 출력하시오.
- 4-2.
  application-prod.yaml에 api.payment.timeout 키가 없으면 경고 문구를 출력하시오.
- 4-3.
  개발(dev)과 운영(prod)의 spring.datasource.url이 같으면 경고를 출력하시오.
- 4-4.
  운영 환경의 jwt.secret이 16자 미만이면 에러를 내고 종료하시오.
- 4-5.
  두 파일의 모든 차이점을 주석 없이 config_diff.txt에 저장하고 완료 시간을 기록하시오.

## 풀이
- 환경 세팅
  실습 파일 자동 생성 스크립트
  ```
  mkdir -p config

cat <<EOF > config/application-dev.yaml
server:
  port: 8080
spring:
  datasource:
    url: jdbc:mysql://dev-db.internal:3306/mydb
    username: dev_user
    password: dev_password
api:
  payment:
    endpoint: https://dev-payment.api.com
    timeout: 3000
logging:
  level:
    root: DEBUG
jwt:
  secret: "dev-secret-key-1234567890"
  expiration: 3600
EOF

cat <<EOF > config/application-prod.yaml
server:
  port: 8080
spring:
  datasource:
    url: jdbc:mysql://prod-db.internal:3306/mydb
    username: prod_user
    password: prod_password
api:
  payment:
    endpoint: https://payment.api.com
logging:
  level:
    root: INFO
jwt:
  secret: "short"
EOF

echo "✅ 실습 환경 준비 완료! 'config' 폴더를 확인하세요."
```

- Step 1 : 설정 파일 탐색 및 구조 확인
  [풀이]
  ```
  # 1. 파일 찾기
  find config/ -name "application-*.yaml" -o -name "application-*.yml"
  
  # 2. 구조 확인
  yq -o=props config/application-prod.yaml
  ```
  [설명]
  ```
  yq -o=props
  ```
  - YAML의 계층 구조를 a.b.c. = value 형태의 Properties 포맷으로 출력. 
복잡한 들여쓰기를 한 줄의 텍스트로 변환하여 구조 파악이 쉽게 만들어줌.

- Step 2 : 특정 환경(Prod)의 설정 누락 여부 검사
  [풀이]
  ```
  [ "$(yq '.api.payment.timeout' config/application-prod.yaml)" == "null" ] && echo "WARNING: Timeout setting missing!"
  ```
  [설명]
  ```
  yq '.api.payment.timeout'
  ```
  - 점(.)으로 구분된 경로를 따라가 해당 값을 가져옵니다. 만약 해당 경로에 데이터가 없으면 yq는 문자열 null을 반환합니다.
  ```
  $( ... )
  ```
  - 명령어 치환으로 괄호 안의 실행 결과를 텍스트로 가져와 비교문에 넣습니다.
  ```
  [ A == B ] && C
  ```
  - 단축 평가 방식으로 앞의 조건( [ … ] )이 참이면 && 뒤의 명령어를 실행합니다.
- Step 3 : 환경 간 데이터베이스 URL 불일치 리포트 생성
  [풀이]
  ```
DEV_URL=$(yq '.spring.datasource.url' config/application-dev.yaml)
PROD_URL=$(yq '.spring.datasource.url' config/application-prod.yaml)

if [ "$DEV_URL" == "$PROD_URL" ]; then
    echo "🚨 CRITICAL: Dev DB used in Prod!"
fi
```
[설명]
```
VAR=$(...)
```
- yq로 추출한 DB 접속 주소를 변수에 저장합니다.
```
if [ "$A" == "$B" ; then
```
- 두 변수의 값이 일차하는지 비교한 뒤, 운영 서버 설정 파일에 실수로 개발용 DB 주소가 적혀있는 사고를 막는 로직입니다.
- 참고
  - 변수를 따옴표(”)로 감싸는 이유는 값이 비어있을 때 발생할 수 있는 구문 오류를 방지하기 위함입니다.

- Step 4 : 보안 준수 사항 검사 (Secret Key 검증)
  
  [풀이]
  ```
PROD_SECRET=$(yq '.jwt.secret' config/application-prod.yaml)
if [ ${#PROD_SECRET} -lt 16 ]; then
    echo "❌ ERROR: JWT Secret is too short."
    exit 1
fi
```
[설명]
```
${PROD_SECRET}
```
- shell script의 내장 기능으로, 변수에 담긴 문자열의 길이를 반환합니다. 
```
exit 1
```
- 스크립트를 비정상 종료 상태 코드로 마칩니다. CI/CD 파이프라인에서 이 명령어를 만나면 배포가 즉시 중단됩니다.
- Step 5 : 최종 검수 자동화 및 리포트 파일 저장
  [풀이]
  ```
# 1. 평탄화된 임시 파일 생성
yq -o=props config/application-dev.yaml > dev.tmp
yq -o=props config/application-prod.yaml > prod.tmp

# 2. 비교 결과 저장
echo "--- Configuration Diff Report ---" > config_diff.txt
diff -u dev.tmp prod.tmp >> config_diff.txt || true

# 3. 마무리
echo "검수 완료 시간: $(date)" >> config_diff.txt
rm *.tmp
```
[설명]
```
diff -u
```
- Unified diff 모드로, 바뀐 부분만 보여주는게 아니라, 변경된 줄의 앞뒤 문맥을  + (추가), - (삭제) 기호와 함께 보여주어 가독성이 매우 뛰어납니다.
```
``
|| true
```

  
  
  
  
- 
  - 지정한 폴더에서 조건에 맞는 파일들만 솎아내는 작업
    ```
    find practice_env/etc practice_env/opt/app/config practice_env/home/dev/scripts -type f -mtime -14 \( -name "*.conf" -o -name "*.env" -o -name "*.yml" -o -name "*.yaml" -o -name "*.sh" \)
    ```
  [명령어 설명]
     ```
     find practice_env/etc practice_env/opt/app/config practice_env/home/dev/scripts
     ```
  -검색을 시작할 3개의 경로를 한꺼번에 지정
     ```
      -type f
     ```
  - 디렉토리는 제외하고 순수한 파일만 찾기
     ```
     -mtime -14
     ```
  - 수정된 시간이 최근 14일 이내인 파일만 찾기
    ```
      -name "*.conf" -o -name "*.env" ...
     ```
- 파일 이름의 조건을 주어 -o는 OR라는 뜻으로 나열된 확장 중 하나라도 일치하는 파일들만 찾기
- 5-2. : 민감 정보 패턴 감지 (grep + 정규표현식)
  [정답]
  - 찾아낸 파일들의 내부 텍스트를 열어보고, 위험한 패턴이 있는지 검사
    ```
    find practice_env/etc practice_env/opt/app/config practice_env/home/dev/scripts -type f -mtime -14 \( -name "*.conf" -o -name "*.env" -o -name "*.yml" -o -name "*.yaml" -o -name "*.sh" \) -exec grep -iHnE "password=|passwd=|SECRET_KEY|API_KEY|token|AKIA[0-9A-Z]{16}" {} +
    ```
  [명령어 설명]
     ```
     -exec [명령어] {} +
     ```
  - 앞에서 find로 찾은 파일들의 목록을 {}에 모은 후, 뒤에 적힌 grep 명령어를 한 번에 실행시킴
     ```
     AKIA[0-9A-Z]{16}
     ```
  - AWS Access Key를 찾는 정규표현식. AKIA로 시작하고 그 뒤에 숫자나 대문자 영어가 정확히 16글자 오는 문자열을 뜻함
  - 5-3. : 오탐지 제거 - 주석 제외 (grep -v)
    [정답]
    - 앞의 결과물을 파이프( | )로 넘겨받아, 실제로 코드에 반영되지 않는 주석을 제거
       ```
       find practice_env/etc practice_env/opt/app/config practice_env/home/dev/scripts -type f -mtime -14 \( -name "*.conf" -o -name "*.env" -o -name "*.yml" -o -name "*.yaml" -o -name "*.sh" \) -exec grep -iHnE "password=|passwd=|SECRET_KEY|API_KEY|token|AKIA[0-9A-Z]{16}" {} + 2>/dev/null | grep -vE "^[[:space:]]*#"
       ```
    [명령어 설명]
       ```
       2>/dev/null
       ```
    - 에러 메시지를 숨기는 용으로 권한이 없어서 못 읽는 파일이 있을 때 또는 Permission denied 에러로를 모아서 휴지통(/dev/null)으로 버림. 결과 화면을 깨끗하게 유지해주기 위함.
       ```
       |
       ```
    - 파이프 : 앞 명령어의 출력 결과를 뒤 명령어의 입력으로 토스
       ```
       grep -v
       ```
    - 매칭되는 것을 찾는 게 아니라, 반대로 매칭되는 것을 결과에서 제외
       ```
       "^[[:space:]]*#"
       ```
    -주석을 의미하는 정규표현식
      -  ^ : 줄의 시작을 의미
      -  [[:space”]]* : 띄어쓰기 공백이나 탭이 0개 이상 있을 수 있다는 뜻
      -  #: 쉘 스크립트나 설정 파일의 주석 기호
- 5-4.  : 리포트 포맷팅 (awk)
  [정답]
  - 파이프라인의 최종 목적지로, 지저분한 텍스트를 원하는 형태의 리포트로 조립
    ```
    find practice_env/etc practice_env/opt/app/config practice_env/home/dev/scripts -type f -mtime -14 \
    \( -name "*.conf" -o -name "*.env" -o -name "*.yml" -o -name "*.yaml" -o -name "*.sh" \) -exec \
    grep -iHnE "password=|passwd=|SECRET_KEY|API_KEY|token|AKIA[0-9A-Z]{16}" {} + 2>/dev/null | \
    grep -vE "^[[:space:]]*#" | \
    awk -F':' '{
        content = substr($0, length($1) + length($2) + 3)
        print "[위험탐지] 파일: " $1 " | 라인: " $2 " | 내용: " content
    }' > secret_scan_report.txt
    ```
[명령어 설명]
   ```
   awk -F':'
   ```
   - 구분자를 콜론(:)으로 설정하여 텍스트를 쪼갬
   ```
   content = substr($0, length($1) + length($2) + 3)
   ```
   - substr 함수를 사용하여 해당 줄의 전체 내용($0)에서 “파일명 길이 + 줄번호 길이 + 콜론 2개 길이’만큼을 건너뛰고 나머지 뒷부분을 통째로 잘라옴
   ```
   > secret_scan_report.txt
   ```
   - awk가 화면에 뿌리려던 내용을 텍스트 파일에 덮어쓰기로 저장



  
    
