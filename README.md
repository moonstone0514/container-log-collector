# container-log-collector



# 🐳 Docker Log Collector (with Bash + Cron + Awk)

이 프로젝트는 **Filebeat 같은 무거운 에이전트 없이**,  
단순히 `bash`, `cron`, `awk` 만을 이용해서 **Docker 컨테이너 오류 로그를 중앙에서 수집·분류·모니터링**하는 경량 시스템입니다.  

---

## 📌 기능 요약

- ✅ 컨테이너별 로그 자동 수집 (offset 기반, 1분 주기)
- ✅ 에러 패턴 필터링 (`error`, `failed`, `crash`, `oomkilled`)
- ✅ 매 정각마다 리포트 생성 (키워드별 텍스트 파일)
- ✅ `tail -F` 기반 실시간 모니터링 지원
- ✅ 크론탭/시스템드 타이머로 자동화 가능

---

## 📂 디렉토리 구조

```

docker-log-project/
├── container-log-collect.sh   # 1분마다 실행 (크론탭)
├── container-log-split.sh     # 정각 실행, 리포트 생성
├── monitor.sh                 # 실시간 모니터링 (tail -F)
├── offsets/                   # 컨테이너별 offset 기록
├── logs/                      # 수집된 로그 저장
└── reports/                   # 시간/날짜별 리포트

````

---

## 🚀 설치 & 실행




### 1. 저장소 클론
```bash
git clone https://github.com/<username>/docker-log-project.git
cd docker-log-project
````

### 2. 실행 권한 부여

```bash
chmod +x container-log-collect.sh container-log-split.sh monitor.sh
```

### 3. 크론탭 등록

```bash
crontab -e
```

아래 내용 추가:

```cron
# 매 1분마다 로그 수집
* * * * * /home/ubuntu/docker-log-project/container-log-collect.sh

```


```
BASE_DIR="/home/ubuntu/docker-log-project"     # 프로젝트 기본 디렉토리
OFFSET_DIR="$BASE_DIR/offsets"                 # 컨테이너별 offset 저장 디렉토리
LOG_DIR="$BASE_DIR/logs"                       # 수집된 로그 저장 디렉토리
TODAY=$(date +%F)                              # 오늘 날짜 (YYYY-MM-DD 형식)

# 디렉토리 생성 (없으면 자동 생성)
mkdir -p "$OFFSET_DIR" "$LOG_DIR"

# 실행 중인 모든 컨테이너 순회
for cid in $(docker ps -q); do
    # 컨테이너 전체 ID (64자짜리)
    fullid=$(docker inspect --format '{{.Id}}' $cid)
    # 컨테이너 이름 (앞의 / 제거)
    cname=$(docker inspect --format '{{.Name}}' $cid | sed 's#^/##')

    # 컨테이너 로그 파일 경로 (json-file 로그 드라이버 기준)
    clog="/var/lib/docker/containers/$fullid/${fullid}-json.log"

    # 오늘 날짜 기준 아웃풋 로그 파일
    outfile="$LOG_DIR/${cname}-all-$TODAY.log"

    # offset 기록 파일 (마지막으로 읽은 바이트 위치 저장)
    offset_file="$OFFSET_DIR/${fullid}.offset"

    # 처음 실행 시 offset 파일이 없으면 현재 로그 파일 크기를 저장
    # → 이미 있는 이전 로그는 건너뛰고 새로 쌓이는 부분부터 수집 시작
    if [ ! -f "$offset_file" ]; then
        wc -c < "$clog" > "$offset_file"
    fi

    # 마지막으로 읽은 위치(바이트 단위) 불러오기
    last_offset=$(cat "$offset_file")

    # 로그 파일에서 last_offset 이후의 새로운 데이터만 tail로 읽기
    tail -c +$((last_offset+1)) "$clog" \
      | awk '{
          # 각 줄에 수집 시각을 붙여서 출력
          cmd="date +\"[%Y-%m-%d %H:%M:%S]\""
          cmd | getline ts
          close(cmd)
          print ts, $0
        }' >> "$outfile"

    # 로그 파일의 현재 전체 크기를 offset으로 갱신
    # → 다음 실행 때는 여기서부터 이어서 읽음
    new_offset=$(wc -c < "$clog")
    echo "$new_offset" > "$offset_file"
done
```

```
# 매 정각 리포트 생성
59 23 * * * /home/ubuntu/docker-log-project/container-log-report.sh
```

```
#!/bin/bash
BASE="$HOME/docker-log-project"
LOGDIR="$BASE/logs"
REPORTDIR="$BASE/reports"
TODAY=$(date +%F)
DST="$REPORTDIR/$TODAY"
mkdir -p "$DST"
# 오늘자 모든 로그 합치기
cat $LOGDIR/*-error-$TODAY.log > "$DST/all-errors.log"
# 에러 키워드별 분리
errors=("Error" "Failed" "CrashLoopBackOff" "OOMKilled")
for err in "${errors[@]}"; do
    grep -i "$err" "$DST/all-errors.log" > "$DST/${err}.txt"
done
```

### 4. 실시간 모니터링

```bash
./monitor.sh
```

---

## 📊 리포트 예시

<img width="916" height="740" alt="image" src="https://github.com/user-attachments/assets/8b7a6cec-dd17-4e90-a3eb-16d4eb201514" />


---

## 🧪 테스트 방법

### 컨테이너 오류 만들기

```bash
# 잘못된 이미지 실행 → 이미지 에러
docker run --name noimg-test doesnotexist:latest

# 메모리 제한 → OOMKilled
docker run -m 50m --memory-swap 50m --name oom-test busybox sh -c "dd if=/dev/zero of=/dev/null bs=100M"

# 강제 종료 → Exit(137)
docker run -d --name crash-nginx nginx
docker kill -s SIGKILL crash-nginx
```

---

## 🛠️ 트러블슈팅

### 중복 로그 저장 문제
처음에는 `container-log-collect.sh`를 크론으로 분 단위 실행했을 때,  
같은 로그가 여러 번 중복 저장되는 문제가 있었습니다.  

이유는 각 실행마다 `docker logs`가 이전 로그까지 모두 다시 읽어오기 때문입니다.  

### 해결 방법: 컨테이너별 Offset 관리
- 컨테이너별로 마지막으로 읽은 타임스탬프(`offsets/CONTAINER_ID.last`)를 기록  
- 다음 실행 시 `--since <last_timestamp>` 옵션을 적용하여,  
  이전에 수집한 로그 이후의 부분만 가져오도록 개선했습니다.  

이를 통해 크론 기반 주기 실행에서도 **중복 없이 로그를 수집**할 수 있게 되었습니다.

