# 🐳 Docker Log Collector (Bash + Cron + Awk)

---

## 🎯 프로젝트 목적

현대 서비스 환경에서는 수십~수백 개의 컨테이너가 동시에 실행되며,  
문제가 발생했을 때 **어떤 컨테이너에서 어떤 유형의 에러가 발생했는지**를 빠르게 파악하는 것이 매우 중요합니다.  

하지만 Docker 기본 로그만으로는:
- 컨테이너별로 흩어져 있어 한눈에 보기 어렵고
- 에러/경고/정보 메시지가 뒤섞여 있어 중요한 로그를 놓치기 쉽습니다.

이 프로젝트는 이러한 문제를 해결하기 위해 만들어졌습니다.  

1. **중앙 집중 수집**  
   - 모든 컨테이너의 오류 로그를 한 곳에 모읍니다.  
   - 크론 기반으로 주기적으로 수집하여 최신 상태를 유지합니다.  

2. **중복 없는 관리**  
   - offset 기반으로 직전 실행 이후 로그만 가져오므로,  
     크론 실행에도 같은 로그가 반복 저장되지 않습니다.  

3. **에러 패턴별 분류**  
   - `Error`, `Failed`, `CrashLoopBackOff`, `OOMKilled` 등 주요 패턴을 자동으로 필터링해  
     별도 리포트 파일로 정리합니다.  
   - 이를 통해 운영자는 원하는 유형의 문제만 빠르게 찾아볼 수 있습니다.  

👉 요약하면, 이 프로젝트의 목적은 **컨테이너 운영 환경에서 오류 로그를 자동으로 수집하고,  
중복 없이, 유형별로 구조화하여 빠른 문제 분석과 대응을 가능하게 하는 것**입니다.


---

## 📌 주요 기능

* ✅ **컨테이너별 로그 자동 수집** (offset 기반, 1분 주기)
* ✅ **에러 패턴 필터링** (`error`, `failed`, `crash`, `oomkilled`)
* ✅ **리포트 자동 생성** (매일 지정된 시간에 키워드별 분리)
* ✅ **실시간 모니터링 지원**
* ✅ **경량 운영** (Bash + Cron + Awk 만 사용)

---

## 📂 디렉토리 구조
<img width="886" height="727" alt="image" src="https://github.com/user-attachments/assets/8181bfbd-e7a5-4d6d-8aea-857f5ba50d4a" />

---

## 🚀 설치 및 실행

### 1. 저장소 클론

```bash
git clone https://github.com/<username>/docker-log-project.git
cd docker-log-project
```

### 2. 실행 권한 부여

```bash
chmod +x container-log-collect.sh container-log-report.sh monitor.sh
```

### 3. 크론탭 등록

```bash
crontab -e
```

아래 내용 추가:

```cron
# 매 1분마다 로그 수집
* * * * * /home/ubuntu/docker-log-project/container-log-collect.sh

# 매일 12:20에 리포트 생성
20 12 * * * /home/ubuntu/docker-log-project/container-log-report.sh
```

---

## ⚙️ 수집 스크립트 예시 (`container-log-collect.sh`)

```bash
BASE_DIR="/home/ubuntu/docker-log-project"
OFFSET_DIR="$BASE_DIR/offsets"
LOG_DIR="$BASE_DIR/logs"
TODAY=$(date +%F)

mkdir -p "$OFFSET_DIR" "$LOG_DIR"

for cid in $(docker ps -q); do
    fullid=$(docker inspect --format '{{.Id}}' $cid)
    cname=$(docker inspect --format '{{.Name}}' $cid | sed 's#^/##')
    clog="/var/lib/docker/containers/$fullid/${fullid}-json.log"

    outfile="$LOG_DIR/${cname}-all-$TODAY.log"
    offset_file="$OFFSET_DIR/${fullid}.offset"

    # 초기 실행: 현재 파일 크기 기록
    if [ ! -f "$offset_file" ]; then
        wc -c < "$clog" > "$offset_file"
    fi

    last_offset=$(cat "$offset_file")

    # 새로운 데이터만 읽기
    tail -c +$((last_offset+1)) "$clog" \
      | awk '{
          cmd="date +\"[%Y-%m-%d %H:%M:%S]\""
          cmd | getline ts; close(cmd)
          print ts, $0
        }' >> "$outfile"

    # offset 갱신
    wc -c < "$clog" > "$offset_file"
done
```

---

## 📑 리포트 스크립트 예시 (`container-log-report.sh`)

```bash
BASE="$HOME/docker-log-project"
LOGDIR="$BASE/logs"
REPORTDIR="$BASE/reports"
TODAY=$(date +%F)
DST="$REPORTDIR/$TODAY"

mkdir -p "$DST"

# 오늘자 로그 합치기
cat $LOGDIR/*-all-$TODAY.log > "$DST/all.log"

# 에러 키워드별 분류
errors=("Error" "Failed" "CrashLoopBackOff" "OOMKilled")
for err in "${errors[@]}"; do
    grep -i "$err" "$DST/all.log" > "$DST/${err}.txt"
done
```

---

## 👀 실시간 모니터링 (`monitor.sh`)

```bash
#!/bin/bash
tail -F logs/*-all-$(date +%F).log
```
<img width="861" height="209" alt="image" src="https://github.com/user-attachments/assets/a9f7e449-a20c-4f70-b674-571e7bee2c48" />

---

## 📊 리포트 결과 예시

```
reports/2025-09-06/
├── all.log
├── Error.txt
├── Failed.txt
├── CrashLoopBackOff.txt
└── OOMKilled.txt
```

---

## 🧪 테스트 방법

### 1. 잘못된 이미지 실행 → 이미지 에러

```bash
docker run --name noimg-test doesnotexist:latest
```

### 2. 메모리 제한 → OOMKilled

```bash
docker run -m 50m --memory-swap 50m --name oom-test \
  busybox sh -c "dd if=/dev/zero of=/dev/null bs=100M"
```

### 3. 강제 종료 → Exit(137)

```bash
docker run -d --name crash-nginx nginx
docker kill -s SIGKILL crash-nginx
```

---

## 🛠️ 트러블슈팅

### 중복 로그 저장 문제

* 초기에 `docker logs` 사용 시, 실행할 때마다 전체 로그가 중복 수집되는 문제가 있었음.

### 해결 방법

* **컨테이너별 offset 관리**:

  * 각 컨테이너의 로그 파일 크기(바이트 단위)를 기록
  * 다음 실행 시 해당 offset 이후만 수집
  * → 중복 없는 로그 수집 가능
