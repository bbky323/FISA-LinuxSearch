## 🛡️상황 가정

신규 서비스 배포 전에 DevOps 팀이 Kubernetes YAML 매니페스트를 검토 중이다.

그런데 운영 정책상 반드시 들어가야 하는 값들이 빠진 경우가 종종 있어 수동 점검이 필요하다.

---

## ❓문제

검수 스크립트 만들기

```jsx
nano check_k8s.sh
```

deployment.yaml, service.yaml, configmap.yaml  등을 검토하여 다음을 점검하시오.

- 모든 YAML 문법이 올바른지 확인
- Deployment 리소스의 replicas 값이 2 이상인지 확인
- 컨테이너 이미지 태그가 latest인지 검사
- resources.requests와  resources.limits가 모두 설정돼 있는지 확인
- env에 민감정보가 평문으로 직접 들어간 항목이 있는지 점검
- 결과를 k8s_manifest_report.txt에 저장

### deployment.yaml

```jsx
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-app
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: app
          image: nginx:latest
          resources:
            requests:
              cpu: "100m"
            limits:
              memory: "256Mi"
          env:
            - name: PASSWORD
              value: "1234"
```

### service.yaml

```jsx
apiVersion: v1
kind: Service
metadata:
  name: test-service
spec:
  selector:
    app: test
```

### configmap.yaml

```jsx
apiVersion: v1
kind: ConfigMap
metadata:
  name: test-config
data:
  DB_HOST: localhost
```

---

## 💡풀이

## 전체 스크립트

```
#!/bin/bash

OUTPUT="k8s_manifest_report.txt"
>$OUTPUT

echo"===== Kubernetes YAML 검수 결과 =====" >>$OUTPUT

# 1. YAML 문법 검사
echo"[문법 검사]" >>$OUTPUT
yamllint deployment.yamlservice.yaml configmap.yaml >>$OUTPUT2>&1

# 2. replicas 검사
echo-e"\n[replicas 검사]" >>$OUTPUT
replicas=$(yq'.spec.replicas' deployment.yaml)
if ["$replicas"-lt2 ];then
echo"❌ replicas 값이 2 미만:$replicas" >>$OUTPUT
else
echo"⭕ replicas 정상:$replicas" >>$OUTPUT
fi

# 3. latest 태그 검사
echo-e"\n[이미지 태그 검사]" >>$OUTPUT
image=$(yq'.spec.template.spec.containers[].image' deployment.yaml)
ifecho"$image" |grep-q":latest";then
echo"❌ latest 태그 사용:$image" >>$OUTPUT
else
echo"⭕ 이미지 태그 정상:$image" >>$OUTPUT
fi

# 4. resources 검사
echo-e"\n[리소스 설정 검사]" >>$OUTPUT
req=$(yq'.spec.template.spec.containers[].resources.requests' deployment.yaml)
lim=$(yq'.spec.template.spec.containers[].resources.limits' deployment.yaml)

if ["$req"="null" ] || ["$lim"="null" ];then
echo"❌ resources 설정 누락" >>$OUTPUT
else
echo"⭕ resources 설정 존재" >>$OUTPUT
fi

# 5. env 평문 민감정보 검사
echo-e"\n[환경변수 보안 검사]" >>$OUTPUT
yq'.spec.template.spec.containers[].env[]' deployment.yaml |grep-i"password\|secret" >> temp_env.txt

if [-s temp_env.txt ];then
echo"❌ 민감정보 평문 포함:" >>$OUTPUT
cat temp_env.txt >>$OUTPUT
else
echo"⭕ 민감정보 없음" >>$OUTPUT
fi

rm-f temp_env.txt

echo-e"\n===== 검사 완료 =====" >>$OUTPUT
```

## 📊정답 결과 <img width="1632" height="338" alt="image" src="./mission1_2.png" />