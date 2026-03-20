#  ❓문제 1 - 평문 비밀번호 노출 여부 검사

## 📚가정 상황
- 개발자가 제보한 "평문 비밀번호 노출 의심" 건을 조사해야 합니다. 
- 전체 서버를 뒤지면 시간이 너무 오래 걸리므로, 특정 디렉토리의 최근 수정된 설정/스크립트 파일만 추려내어 정규표현식으로 위험 패턴을 색출하는 파이프라인을 구축하시오.

## 🤔문제
### 5-1.
   > /etc, /opt/app/config, /home/dev/scripts 디렉토리에서 최근 14일 이내에 수정된 파일 중 확장자가 .conf, .env, .yml, .yaml, .sh인 파일만 검색하는 find 명령어를 작성하시오.
### 5-2.
  > 5-1. 에서 찾은 파일들을 대상으로, 대소문자 구분 없이 아래 패턴 중 하나라도 포함된 라인을 찾으시오. (결과에 파일명과 라인 번호가 출력되어야 함)<br>
  탐지 패턴: `password=`, `passwd=`, `SECRET_KEY`, `API_KEY`, `token`, AWS Access Key 형식(`AKIA`로 시작하는 20자리 영문대문자/숫자)
### 5-3.
  > 5-2. 의 결과 중, 코드 맨 앞이 #으로 시작하는 주석 처리된 라인은 실제 실행되지 않으므로 결과에서 제외하시오. <br>
  (단, # 앞에 들여쓰기 공백이나 탭이 있는 경우도 주석으로 간주하여 걸러내야 함)
### 5-4.
  > 최종 필터링된 결과를 `awk`를 사용하여 아래의 규격화된 포맷으로 변환하고, `secret_scan_report.txt` 파일로 저장하시오.
  출력 포맷: `[위험탐지] 파일: <경로> | 라인: <줄번호> | 내용: <매칭된 내용>`

## 📝풀이
### 환경 세팅
  - 1. setup_dummy.sh 파일 생성
    ```
    vi setup_dummy.sh
    ```
    ```
    #!/bin/bash

    # 실습용 최상위 가상 디렉토리 설정 (현재 위치에 생성)
    BASE_DIR="./practice_env"
  
    echo "=== 실습 환경 구성을 시작합니다 ==="
     
    # 1. 가상 디렉토리 구조 생성
    mkdir -p "$BASE_DIR/etc"
    mkdir -p "$BASE_DIR/opt/app/config"
    mkdir -p "$BASE_DIR/home/dev/scripts"
     
    # 2. 더미 파일 및 데이터 생성
     
    # [케이스 A] 정상적으로 탐지되어야 하는 파일들
    cat <<EOF > "$BASE_DIR/opt/app/config/database.yml"
    db_host: localhost
    db_user: admin
    # 아래는 주석 처리된 비밀번호 (탐지에서 제외되어야 함!)
    # password=old_password
       # passwd=spaced_comment_pass
     
    # 진짜 비밀번호 (탐지되어야 함!)
    password=super_secret_p@ssw0rd!
    EOF
     
    cat <<EOF > "$BASE_DIR/home/dev/scripts/deploy.sh"
    #!/bin/bash
    echo "Deploying..."
    # AWS 키 설정 
    export AWS_ACCESS_KEY="AKIAIOSFODNN7EXAMPLE"
    SECRET_KEY="my_secret_key_123"
    EOF
     
    cat <<EOF > "$BASE_DIR/home/dev/scripts/.env"
    APP_ENV=production
    # token=old_token
    token=ey1234567890
    EOF
     
    # [케이스 B] 확장자가 달라서 제외되어야 하는 함정 파일 (.txt)
    cat <<EOF > "$BASE_DIR/home/dev/scripts/memo.txt"
    API_KEY=this_should_not_be_found
    EOF
     
    # [케이스 C] 14일 이전 파일이라 제외되어야 하는 함정 파일 (mtime 조작)
    cat <<EOF > "$BASE_DIR/etc/old_config.conf"
    password=very_old_password
    EOF
    # 수정 시간을 20일 전으로 조작 (touch 명령어 활용)
    touch -d "20 days ago" "$BASE_DIR/etc/old_config.conf"
     
    echo "=== 실습 환경 구성이 완료되었습니다! ==="
    echo "생성된 위치: $BASE_DIR"
    ```
- 2. setup_dummy.sh 파일 실행 권한 부여
   ```
   chmod +x setup_dummy.sh
   ```
- 3. setup_dummy.sh 파일 실행
   ```
   ./setup_dummy.sh
   ```
- 4.폴더 구조
   ```
   ubuntu@server01:~/problem5$ tree practice_env
   practice_env
   ├── etc
   │   └── old_config.conf
   ├── home
   │   └── dev
   │       └── scripts
   │           ├── deploy.sh
   │           └── memo.txt
   └── opt
       └── app
           └── config
               └── database.yml
   ```
### 5-1. 대상 파일 추려내기 (find 기초)
  > [정답]
    - 지정한 폴더에서 조건에 맞는 파일들만 솎아내는 작업
      ```
      find practice_env/etc practice_env/opt/app/config practice_env/home/dev/scripts -type f -mtime -14 \( -name "*.conf" -o -name "*.env" -o -name "*.yml" -o -name "*.yaml" -o -name "*.sh" \)
      ```
  > [명령어 설명]
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
### 5-2. : 민감 정보 패턴 감지 (grep + 정규표현식)<br>
  > [정답]
    - 찾아낸 파일들의 내부 텍스트를 열어보고, 위험한 패턴이 있는지 검사
    ```
    find practice_env/etc practice_env/opt/app/config practice_env/home/dev/scripts -type f -mtime -14 \( -name "*.conf" -o -name "*.env" -o -name "*.yml" -o -name "*.yaml" -o -name "*.sh" \) -exec grep -iHnE "password=|passwd=|SECRET_KEY|API_KEY|token|AKIA[0-9A-Z]{16}" {} +
    ```
  > [명령어 설명]
    ```
     -exec [명령어] {} +
    ```
    - 앞에서 find로 찾은 파일들의 목록을 {}에 모은 후, 뒤에 적힌 grep 명령어를 한 번에 실행시킴
    ```
     AKIA[0-9A-Z]{16}
    ```
    - AWS Access Key를 찾는 정규표현식. AKIA로 시작하고 그 뒤에 숫자나 대문자 영어가 정확히 16글자 오는 문자열을 뜻함
### 5-3. : 오탐지 제거 - 주석 제외 (grep -v)
  > [정답]
    - 앞의 결과물을 파이프( | )로 넘겨받아, 실제로 코드에 반영되지 않는 주석을 제거
    ```
      find practice_env/etc practice_env/opt/app/config practice_env/home/dev/scripts -type f -mtime -14 \( -name "*.conf" -o -name "*.env" -o -name "*.yml" -o -name "*.yaml" -o -name "*.sh" \) -exec grep -iHnE "password=|passwd=|SECRET_KEY|API_KEY|token|AKIA[0-9A-Z]{16}" {} + 2>/dev/null | grep -vE "^[[:space:]]*#"
    ```
  > [명령어 설명]
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
### 5-4.  : 리포트 포맷팅 (awk)
  > [정답]
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
  > [명령어 설명]
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

## ✅정답 결과

<img width="1632" height="338" alt="image" src="./mission1_1.png" />
