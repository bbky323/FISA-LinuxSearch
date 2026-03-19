## 🛡️상황 가정

배포 직전, 보안팀이 “설정 파일과 샘플 데이터 파일에 비밀번호, API 키, 토큰이 평문으로 남아 있지 않은지” 최종 점검하라고 요청했다.

---

## ❓문제

프로젝트 디렉토리 아래의 ```.json, .yaml, .yml``` 파일을 대상으로 다음을 수행하시오.

### 요구사항

1. **```password, passwd, secret, token, apiKey``` 값이 하드 코딩 되었거나 비어있는 JSON/YAML 파일을 찾기**

    - 결과를 hardcoded_secrets.txt 에 저장

2. **문법 오류가 있는 JSON/YAML 파일을 찾아 저장**

    - syntax_errors.txt에 저장

---

## 💡풀이

### 1. 테스트 데이터셋 생성 스크립트 작성(setup_audit.sh)

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

### 2. 보안 감사 스크립트 작성(run_audit.sh)


```jsx
#!/bin/bash

SYNTAX_LOG="syntax_errors.txt"
SECRET_LOG="hardcoded_secrets.txt"

> "$SYNTAX_LOG"
> "$SECRET_LOG"

# 검색 대상 키
TARGET_KEYS="password|passwd|secret|token|apiKey"

echo "검수를 시작합니다..."

find . -maxdepth 2 -type f \( -name "*.json" -o -name "*.yaml" -o -name "*.yml" \) | while read -r file; do
    
    [[ "$file" == *"$SYNTAX_LOG"* || "$file" == *"$SECRET_LOG"* ]] && continue

    # [1] JSON 처리 (jq)
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

    # [2] YAML 처리
    elif [[ "$file" == *.yaml || "$file" == *.yml ]]; then
        if ! yq "." "$file" > /dev/null 2>&1; then
            echo "[문법 오류] $file" >> "$SYNTAX_LOG"
        else

            res=$(yq "." "$file" | grep -iE "\"($TARGET_KEYS)\":\s*\"[^\$]" | grep -v "\${")
            
            if [[ -n "$res" ]]; then
                echo "[$file] 발견된 항목:" >> "$SECRET_LOG"
                echo "$res" | sed 's/^[[:space:]]*//' | sed "s/^/  - /" >> "$SECRET_LOG"
                echo "" >> "$SECRET_LOG"
            fi
        fi
    fi
done

echo "검수 완료!"
```

### 3. 스크립트 권한 부여 및 실행
#### 3.1 스크립트 권한 부여
``` chmod +x setup_audit.sh run_audit.sh ```

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


