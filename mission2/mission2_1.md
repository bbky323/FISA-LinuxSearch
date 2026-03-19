### 가정 상황

- 결제 서비스 API를 새 버전으로 교체한 뒤, 프론트엔드 팀이 “응답 JSON 구조가 일부 달라져서 화면이 깨진다”고 보고했다.
- 운영자는 서버에 저장된 API 샘플 응답 파일들을 빠르게 검수해야 한다.

### 문제

- 여러 개의 API 응답 JSON 파일 중에서 다음 조건을 점검하시오.
    - JSON 문법이 올바른 파일만 선별
    - `status`, `userId`, `amount`, `timestamp` 필드가 모두 존재하는지 확인
    - `amount`가 숫자가 아닌 잘못된 응답 파일 찾기
    - `status`가 `"success"`인 응답만 추출
    - 결과를 `api_validation_report.txt`로 저장

### 풀이

### 1단계. JSON 문법이 올바른 파일만 선별하기

> JSON 문법이 올바르게 적용된 파일만 선별하도록 조건을 걸어준다.
> 

```jsx
for f in api_responses/*.json; do 
    if jq -e . "$f" > /dev/null 2>&1; then
        echo "[$f] 문법 정상"
    else
        echo "[$f] 문법 오류 (파싱 실패)"
    fi
done
```



### 2단계. 필수 필드 존재 여부 확인 추가

> `status`, `userId`, `amount`, `timestamp` 필드가 모두 존재하는지 확인하는 조건을 추가한다.
> 

```jsx
for f in api_responses/*.json; do 
    if jq -e . "$f" > /dev/null 2>&1; then
        jq -r '
          "[\(input_filename)] " + 
          if (has("status") and has("userId") and has("amount") and has("timestamp")) == false then " 필수 필드 누락" 
          else " 필드 정상" 
          end
        ' "$f"
    fi
done
```

### 3단계. amount가 숫자인지 확인 (타입 검사)

> `amount`가 숫자가 아닌 잘못된 응답 파일을 찾도록 조건을 추가한다.
> 

```jsx
for f in api_responses/*.json; do 
    if jq -e . "$f" > /dev/null 2>&1; then
        jq -r '
          "[\(input_filename)] " + 
          if (has("status") and has("userId") and has("amount") and has("timestamp")) == false then " 필수 필드 누락" 
          elif (.amount | type) != "number" then " amount 타입 오류" 
          else " 통과" 
          end
        ' "$f"
    fi
done
```

### 4단계. status가 "success"인지 확인

> `status`가 `"success"`인 응답만 추출하도록 조건을 추가한다.
> 

```jsx
for f in api_responses/*.json; do 
    if jq -e . "$f" > /dev/null 2>&1; then
        jq -r '
          "[\(input_filename)] " + 
          if (has("status") and has("userId") and has("amount") and has("timestamp")) == false then " 필수 필드 누락" 
          elif (.amount | type) != "number" then " amount 타입 오류" 
          elif .status != "success" then " 상태 오류 (success 아님)" 
          else " 정상 응답" 
          end
        ' "$f"
    fi
done
```

### 5단계. 결과를 파일로 저장하기

> 결과를 `api_validation_report.txt`로 저장한다.
> 

```jsx
for f in api_responses/*.json; do 
    jq -r '
      "[\(input_filename)] " + 
      if (has("status") and has("userId") and has("amount") and has("timestamp")) == false then "필수 필드 누락" 
      elif (.amount | type) != "number" then "amount 타입 오류" 
      elif .status != "success" then "상태 오류 (success 아님)" 
      else "정상 응답" 
      end
    ' "$f" >> api_validation_report.txt
done
```

| 옵션 | 의미 |
| --- | --- |
| `jq -e` | 문법이 올바른지 검사해서 참/거짓 반환 |
| `jq -r` | 쌍따옴표를 제거하고 텍스트만 출력 |
| `has("key")` | 특정 키가 존재하는지 확인 |
| `type` | 데이터의 타입을 반환 |
| `input_filename` | 현재 처리 중인 파일의 이름을 반환 |
| `+` | 두 문자열을 하나로 합침 |
| `\()` | 문자열 안에 변수를 삽입 |

---

### 정답 결과
<img width="1040" height="405" alt="image" src="https://github.com/user-attachments/assets/8dea60e5-4d64-47f5-b135-6e24fbffdb76" />

