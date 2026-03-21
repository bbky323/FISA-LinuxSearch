# 🔐문제5: Shell 스크립트를 활용한 JSON/YAML 설정 파일 보안 및 문법 자동 감사

## 가정 상황

배포 직전, 보안팀이 “설정 파일과 샘플 데이터 파일에 비밀번호, API 키, 토큰이 평문으로 남아 있지 않은지” 최종 점검하라고 요청했다.

---

## ❓문제

프로젝트 디렉토리 아래의 ```.json, .yaml, .yml``` 파일을 대상으로 다음을 수행하시오.

1. **```password, passwd, secret, token, apiKey``` 값이 하드 코딩 되었거나 비어있는 JSON/YAML 파일을 찾기**

    - 결과를 **hardcoded_secrets.txt** 에 저장

2. **문법 오류가 있는 JSON/YAML 파일을 찾아 저장**

    - 결과를 **syntax_errors.txt**에 저장


#### 테스트 데이터
- setup_audit.sh를 사용하여 총 6개의 json 및 yaml파일 생성(정상 파일 2개, 문법 오류 파일 2개, 키 하드코딩 파일 2개)
```jsx
# 1. 정상 파일 (환경 변수 사용 - 통과 대상)
cat <<EOF > config_safe.json
{
  "db": {
    "user": "admin",
    "password": "\${DB_PASSWORD}",
    "apiKey": "\${CLOUD_KEY}"
  }
}
EOF

cat <<EOF > deploy_safe.yaml
spec:
  containers:
    - name: app
      env:
        - name: SECRET_TOKEN
          value: "\${AUTH_TOKEN}"
EOF

# 2. 문법 오류 파일 (Syntax Error)
echo '{"status": "active", "list": [1, 2, ' > error_incomplete.json
echo -e "services:\n  web:\n    image: nginx\n      bad_indent: true" > error_bad_indent.yaml

# 3. 하드코딩된 민감 정보 파일 (Vulnerable)
cat <<EOF > db_vulnerable.json
{
  "password": "plain_password_1234",
  "apiKey": "AKIA_EXAMPLETODISCOVER"
}
EOF

cat <<EOF > app_vulnerable.yml
auth:
  token: "9a8b7c6d5e4f"
  passwd: "hardcoded_val"
EOF

echo "테스트 데이터 생성 완료 (audit_workspace 디렉토리)"
```
---

## 💡풀이

### 1. 보안 감사 스크립트 작성(run_audit.sh)
- 프로젝트 내의 설정 파일(.json, .yaml, .yml)을 스캔하여 문법 오류 및 소스 코드 내에 직접 작성된 하드코딩된 중요 정보(비밀번호, 토큰 등)를 찾아내는 스크립트

**1.1 로그 파일 지정 및 초기화**

```jsx
#!/bin/bash

SYNTAX_LOG="syntax_errors.txt"
SECRET_LOG="hardcoded_secrets.txt"

> "$SYNTAX_LOG"
> "$SECRET_LOG"

# 검색 대상 키
TARGET_KEYS="password|passwd|secret|token|apiKey"

```

**1.2 대상 파일 탐색**
```jsx
echo "검수를 시작합니다..."

find . -maxdepth 2 -type f \( -name "*.json" -o -name "*.yaml" -o -name "*.yml" \) | while read -r file; do
```
- find 명령어를 사용하여 최대 2단계 하위 디렉토리(-maxdepth 2)까지의 json, yaml, yml 파일들을 찾아 하나씩 검사하기 위해 while 문으로 전달

**1.3 JSON 파일 처리 로직**
```jsx
    if [[ "$file" == *.json ]]; then
        if ! jq . "$file" > /dev/null 2>&1; then
            echo "[문법 오류] $file" >> "$SYNTAX_LOG"
        else
            res=$(jq -r --arg keys "$TARGET_KEYS" 'paths(scalars) as $p | select($p[-1] | test($keys; "i")) | getpath($p) as $v | select($v | test("^\\$\\{.*\\}$") | not) | "\($p | join(".")): \($v)"' "$file")
            if [[ -n "$res" ]]; then
                echo "[$file] 발견된 항목:" >> "$SECRET_LOG"
                echo "$res" | sed "s/^/  - /" >> "$SECRET_LOG"
                echo "" >> "$SECRET_LOG"
            fi
        fi
```
- **문법 검증:**  jq를 사용하여 JSON 파싱을 시도.
- **비밀 값 추출:** jq의 내장 함수를 사용해 키 이름이 **TARGET_KEYS**와 일치하는지 확인. 
  - 일치하는 항목 중 그 값이 환경 변수 형태(```^\\$\\{.*\\}$```)가 아닌 것만 골라내어 로그에 기록.

**1.4 YAML 파일 처리 로직**
```jsx
    # [2] YAML 처리
    elif [[ "$file" == *.yaml || "$file" == *.yml ]]; then
        if ! yq "." "$file" > /dev/null 2>&1; then
            echo "[문법 오류] $file" >> "$SYNTAX_LOG"
        else

            res=$(yq "." "$file" | grep -iE "\"($TARGET_KEYS)\":\s*\"[^\$]" | grep -v "\${")
            
            if [[ -n "$res" ]]; then
                echo "[$file] 발견된 항목:" >> "$SECRET_LOG"
                echo "$res" | sed 's/^[[:space:]]*/  - /' >> "$SECRET_LOG"
                echo "" >> "$SECRET_LOG"
            fi
        fi
    fi
done

echo "검수 완료!"
```
- **문법 검증:** JSON과 마찬가지로 yq "."를 통해 YAML 문법의 유효성을 검사
- **비밀 값 추출:** yq로 읽어들인 결과를 grep 명령어의 정규표현식과 결합하여 타겟 키워드를 찾음. 
  - 값의 시작이 $ 기호가 아닌 경우(하드코딩된 경우)를 필터링하여 기록

### 2. 스크립트 권한 부여 및 실행
#### 3.1 스크립트 권한 부여
``` chmod +x setup_audit.sh run_audit.sh ```
- 쉘 스크립트에 실행권한을 부여

#### 3.2 쉘 스크립트 실행
```jsx
# 데이터 생성
./setup_audit.sh

# 감사 실행
./run_audit.sh 
```

---
## 📊결과
### 1. 키 값이 하드코딩 된 파일을 저장한 hardcoded_secrets.txt 

```jsx
[./app_vulnerable.yml] 발견된 항목:
  - "token": "9a8b7c6d5e4f",
  - "passwd": "hardcoded_val"

[./db_vulnerable.json] 발견된 항목:
  - password: plain_password_1234
  - apiKey: AKIA_EXAMPLETODISCOVER
```

### 2. 문법 에러가 있는 파일을 저장한 syntax_errors.txt 

```jsx
[문법 오류] ./error_incomplete.json
[문법 오류] ./error_bad_indent.yaml
```


