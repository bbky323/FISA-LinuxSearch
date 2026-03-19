#  ❓문제 1 - 장애 발생 후 에러 로그 추적 문제

## 📚가정 상황

- 운영 중인 서버의 디스크 사용량이 90%를 넘어 서비스 중단(Disk Full) 위기에 처했습니다.
- 인프라 관리자는 어느 디렉토리에 용량이 큰 파일이 몰려 있는지 **계층적으로 파악**하고, 임시 파일들이 차지하는 **전체 용량을 계산**하여 보고한 뒤, 이를 **주기적으로 정리**하는 자동화 스크립트를 배포해야 합니다.

## 🤔문제

- `/home/project` 디렉토리를 대상으로 다음 작업을 수행하는 쉘 스크립트 `storage_manager.sh`를 스케줄링에 등록하시오.
    - 디렉토리 구조를 실제 크기와 함께 시각화하고 project_map.txt에 저장
    - .tmp 이거나 .cache 인 파일을 모두 찾아서 temp_files.txt에 저장
    - 위에서 찾은 파일의 전체 개수와 합계 용량을 구해서 cleanup_report.txt에 저장
    - 매주 일요일 새벽 4시에 자동으로 storage_manager.sh를 실행하도록 설정

## 📝풀이
### 1단계. 디렉토리 구조 및 파일 크기 시각화

> 디렉토리 구조를 실제 크기와 함께 시각화하고 project_map.txt에 저장한다.
> 

```bash
tree -h --du -I ".git|node_modules" /home/project > /home/soon/linux_test/project/project_map.txt
```

| 옵션 | 의미 |
| --- | --- |
| `-h` | 파일 크기를 사람이 읽기 편하게 K, M, G 단위로 표시 |
| `--du` | 파일 내용에서 특정 패턴 검색 |
| `-I "패턴"` | 특정 패턴과 일치하는 파일이나 폴더를 출력에서 제외 |

<img width="1123" height="373" alt="image" src="https://github.com/user-attachments/assets/58a30284-abee-4369-9220-5f153eb4f68f" />


---

### 2단계. 정리할 임시 파일 목록 검색

> .tmp 이거나 .cache 인 임시 파일을 모두 찾아서 목록을 temp_files.txt에 저장한다.
> 

```bash
find /home/soon/linux_test/project -type f \( -name "**.tmp" -o -name "**.cache" \) -mmin -2880 > /home/soon/linux_test/temp_files.txt
```

| 옵션 | 의미 |
| --- | --- |
| `-type f` | 디렉토리나 링크가 아닌 일반 파일만 찾도록 제한합니다. |
| `\(\)` | 여러 조건을 하나로 묶어주는 그룹화 기호입니다. (쉘이 오해하지 않게 역슬래시를 붙입니다.) |
| `-mmin -2880` | 최근 2880분(48시간) 이내에 수정된파일 찾기 |
| `-o` | OR연산자 |

<img width="1445" height="173" alt="image" src="https://github.com/user-attachments/assets/cdfb2ce6-3a56-4dc6-87f8-fe93ac952e0c" />


---

### 3단계. 임시 파일의 전체 개수와 합계 용량 구하기

> 2단계에서 찾은 파일의 전체 개수와 합계 용량을 구해서 cleanup_report.txt에 저장한다.
> 

```bash
awk '{
    count++
    cmd = "stat -c%s \"" $0 "\""
    cmd | getline size
    close(cmd)
    sum += size
}
END {
    printf "총 %d개의 임시 파일이 있으며, 합계 용량은 %d Bytes입니다.\n", count, sum
}' /home/soon/linux_test/temp_files.txt > /home/soon/linux_test/cleanup_report.txt
```

| 옵션 | 의미 |
| --- | --- |
| `>awk '{ ... }'` | 입력받은 파일의 각 줄을 하나씩 읽으며 중괄호 안의 작업 실행 |
| `cmd = "stat -c%s \"" $0 "\""` | 현재 줄의 파일 경로를 넣어 파일 용량을 확인하는 쉘 명령어 문자열 만들기 |
| `close(cmd)`  | 실행했던 외부 명령어 종료 |
| `END { ... }`  | 파일의 모든 줄을 읽고 마지막으로 한 번 실행되는 블록 |

<img width="793" height="332" alt="image" src="https://github.com/user-attachments/assets/c6eaf113-f149-445a-b8be-27f3508c1e71" />


---

### 4단계. 스케줄링 등록

> 매주 일요일 새벽 4시에 자동으로 storage_manager.sh를 실행하도록 설정한다.
> 

```bash
0 4 * * 0 /bin/bash /home/soon/linux_test/project/storage_manager.sh >> /home/soon/linux_test/project/storage_manager.log 2>&1
```

---

## ✅정답 결과

- 쉘 스크립트 실행 결과

```jsx
soon@soon:~/linux_test$ cat storage_manager.log
===== Storage Manager 시작 =====
실행 시각: 2026-03-19 15:28:01

[1] project_map.txt 내용
[4.0K]  /home/soon/linux_test/project
├── [4.0K]  logs
│   ├── [ 15M]  query.cache
│   ├── [ 20M]  server.log
│   └── [ 25M]  user_B.tmp
├── [4.0K]  node_modules
│   ├── [ 50K]  index.js
│   ├── [ 30M]  module.cache
│   └── [ 40M]  user_C.tmp
└── [4.0K]  src
    ├── [ 30K]  Main.java
    ├── [ 10M]  min.tmp
    ├── [5.0M]  soon.cache
    └── [ 30K]  Web.java

4 directories, 10 files

[2] temp_files.txt 내용
/home/soon/linux_test/project/node_modules/user_C.tmp
/home/soon/linux_test/project/node_modules/module.cache
/home/soon/linux_test/project/src/min.tmp
/home/soon/linux_test/project/src/soon.cache
/home/soon/linux_test/project/logs/query.cache
/home/soon/linux_test/project/logs/user_B.tmp

[3] cleanup_report.txt 내용
총 6개의 임시 파일이 있으며, 합계 용량은 131072000 Bytes입니다.

[4] 임시 파일 삭제 시작
삭제 완료: /home/soon/linux_test/project/node_modules/user_C.tmp (41943040 Bytes)
삭제 완료: /home/soon/linux_test/project/node_modules/module.cache (31457280 Bytes)
삭제 완료: /home/soon/linux_test/project/src/min.tmp (10485760 Bytes)
삭제 완료: /home/soon/linux_test/project/src/soon.cache (5242880 Bytes)
삭제 완료: /home/soon/linux_test/project/logs/query.cache (15728640 Bytes)
삭제 완료: /home/soon/linux_test/project/logs/user_B.tmp (26214400 Bytes)

[5] 최종 결과
삭제된 파일 수: 6개
확보한 디스크 용량: 131072000 Bytes
===== Storage Manager 종료 =====
```

- 임시 파일 삭제 후 디렉토리 구조
<img width="491" height="265" alt="image" src="https://github.com/user-attachments/assets/289d866d-6bcf-498a-bb51-323fccba8125" />
