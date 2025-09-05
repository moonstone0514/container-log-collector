# 🐳 Docker Log Collector (Bash + Cron + Awk)

이 프로젝트는 **Filebeat 같은 무거운 에이전트 없이**,
순수하게 `bash`, `cron`, `awk` 만으로 **Docker 컨테이너 오류 로그를 수집·분류·모니터링**하는 경량 시스템입니다.

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
