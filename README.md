# 🐳 Docker Log Collector
### 다중 컨테이너 관리를 위한 로그 데이터 수집·분석 시스템

---


### 👥 Team Members
| 강한솔 | 김문석 |
| :---: | :---: |
| [<img width="160px" src="https://github.com/kkangsol.png" />](https://github.com/kkangsol) | [<img width="160px" src="https://github.com/moonstone0514.png" />](https://github.com/moonstone0514) | 
| [@kkangsol](https://github.com/kkangsol) | [@moonstone0514](https://github.com/moonstone0514) |
<br>

---

## 🎯 프로젝트 목적

여러 개의 컨테이너를 동시에 운영하다 보니,  
컨테이너들이 자주 중지되거나, 알지 못하는 사이에 종료되는 경우가 많았습니다.  

하지만 Docker 기본 로그만으로는:
- 컨테이너별로 흩어져 있어 한눈에 보기 어렵고
- 에러/경고/정보 메시지가 뒤섞여 있어 중요한 로그를 놓치기 쉽습니다.

이러한 문제를 빠르게 확인하고 관리하기 위해,  
어떤 컨테이너에서 어떤 유형의 에러가 발생했는지를 자동으로 수집·분석할 수 있는 시스템을 구현했습니다.
 

1. **중앙 집중 수집**  
   - 모든 컨테이너의 오류 로그를 한 곳에 모아 관리합니다.  
   - 크론 기반으로 주기적으로 수집하여 최신 상태를 유지합니다.  
   - 개별 컨테이너에 직접 들어가 로그를 일일이 확인해야 하는 반복적인 수고를 줄여주고,  
     운영자는 단일 뷰에서 전체 시스템의 오류 상황을 한눈에 파악할 수 있습니다.


2. **에러 패턴별 분류**  
   - `Error`, `Failed`, `CrashLoopBackOff`, `OOMKilled` 등 주요 패턴을 자동으로 필터링해  
     별도 리포트 파일로 정리합니다.  
   - 운영자는 원하는 유형의 문제만 빠르게 찾아볼 수 있습니다.

  
3. **경량 운영**  
   - ✅ `Bash`, `Cron`, `Awk` 만을 사용하여 구현되므로 별도의 로그 수집 에이전트(Filebeat, Fluentd 등)를 설치할 필요가 없습니다.  
   - 최소한의 리소스로 동작하기 때문에 운영 환경에 부담을 주지 않으면서도 효과적인 로그 관리가 가능합니다.
     

👉 요약하면, 이 프로젝트의 목적은 **컨테이너 운영 환경에서 오류 로그를 자동으로 수집하고,  
중복 없이, 유형별로 구조화하여 빠른 문제 분석과 대응을 가능하게 하는 것**입니다.

<br>

---

## 📌 주요 기능

* ✅ **컨테이너별 로그 자동 수집** (offset 기반, 1분 주기)
* ✅ **에러 패턴 필터링** (`error`, `failed`, `crash`, `oomkilled`)
* ✅ **리포트 자동 생성** (매일 지정된 시간에 키워드별 분리)
* ✅ **실시간 모니터링 지원**
* ✅ **경량 운영** (Bash, Cron, Awk 만 사용)
<br>

---

## 🗺️프로젝트 구성도

<img width="3004" height="1968" alt="image" src="https://github.com/user-attachments/assets/a14347ac-e408-482d-8c40-2f110c1912f9" />


---

## 📂 프로젝트 수행 구조
<img width="893" height="708" alt="image" src="https://github.com/user-attachments/assets/2f9df981-8368-4000-9a35-a24b7da9b9bc" />



### 🟢 쉘 스크립트
- **`container-log-collect.sh`**  
  - 실행 중인 컨테이너 로그를 주기적으로 수집하여 `logs/` 디렉토리에 저장  
  - 컨테이너별로 날짜 단위 로그 파일 생성  

- **`container-log-report.sh`**  
  - 하루치 수집된 로그를 통합하고, 주요 에러 패턴(`Error`, `Failed`, `CrashLoopBackOff`, `OOMKilled`)을 자동 분류  
  - 결과를 `reports/YYYY-MM-DD/` 아래에 리포트 파일로 저장  

- **`monitor.sh`**  
  - `tail -F` 기반으로 로그를 실시간 모니터링  
  - 테스트 및 장애 대응 시 즉각적으로 로그 확인 가능  

<br>

### 📁 디렉토리
- **`logs/`**  
  - 수집된 컨테이너 로그 저장소 '
  - awk를 활용한 유의미한 데이터 추출출 
  - `*-all-YYYY-MM-DD.log` : 컨테이너별 전체 로그  
  - `*-error-YYYY-MM-DD.log` : 에러만 추출한 로그  

- **`offsets/`**  
  - 각 컨테이너별 로그 수집 상태를 기록하는 내부 관리 디렉토리  

- **`reports/`**  
  - 운영자가 확인할 수 있는 에러 리포트 저장소  
  - `all-errors.log` : 하루치 에러 전체  
  - `Error.txt`, `Failed.txt`, `CrashLoopBackOff.txt`, `OOMKilled.txt` : 유형별 분류 리포트  

<br><br>

---

## 🚀프로젝트 설치 및 실행

### 1. 저장소 클론

```bash
git clone https://github.com/moonstone0514/docker-log-project.git
cd docker-log-project
```
<br>

### 2. 실행 권한 부여

```bash
chmod +x container-log-collect.sh container-log-report.sh monitor.sh
```
<br>

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



### 4. 로그파일 확인

로그는 `logs/` 디렉토리에 일자별로 생성됩니다.  

```bash
ls logs/
````

예시:

```
container-error-2025-09-08.log
container-info-2025-09-08.log
container-warning-2025-09-08.log
```

실시간 로그 확인:

```bash
tail -f logs/container-error-$(date +%F).log
```
<br>

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

## 👀 실시간 모니터링 (`monitor.sh`)

```bash
#!/bin/bash
tail -F logs/*-all-$(date +%F).log
```
<img width="861" height="209" alt="image" src="https://github.com/user-attachments/assets/a9f7e449-a20c-4f70-b674-571e7bee2c48" />



---

## 🧪 테스트 방법

<br>

컨테이너 오류 로그 수집 기능을 검증하기 위해 **의도적으로 오류 상황을 발생**시킬 수 있습니다.  
아래 예시들은 `logs/` 와 `reports/` 디렉토리에 에러 로그가 잘 쌓이는지 확인하기 위한 테스트 시나리오입니다.

<br>


### 1. 존재하지 않는 이미지 Pull 시 에러 발생생
존재하지 않는 이미지를 실행하여, 레지스트리 인증 실패/이미지 없음 에러를 발생시킵니다.

```
docker run --name noimg-test doesnotexist:latest

```
- 기대 로그
   - unauthorized: authentication required
   - pull access denied
- 리포트 위치
   - reports/<오늘날짜>/Error.txt
     
<br>

### 2. 메모리 제한 초과 → OOMKilled
메모리를 강제로 소비해 컨테이너가 Out-Of-Memory(OOM)로 종료되도록 만듭니다.

```
docker run -m 50m --memory-swap 50m --name oom-test \
  busybox sh -c "dd if=/dev/zero of=/dev/null bs=100M"
```
- 기대 로그
   - OOMKilled
   - 종료 상태 코드 137
- 리포트 위치
   - reports/<오늘날짜>/OOMKilled.txt

<br>

### 3. 강제 종료 → Exit(137)
정상 실행 중인 컨테이너를 강제로 SIGKILL 시그널로 종료시킵니다.

```
docker run -d --name crash-nginx nginx
docker kill -s SIGKILL crash-nginx
```
- 기대 로그
   - crash, exited with code 137
- 리포트 위치
   - reports/<오늘날짜>/Failed.txt 또는 CrashLoopBackOff.txt
 
<br>

### ✅ 테스트 후 확인 방법

```
# 오늘자 에러 리포트 디렉토리 확인
ls reports/$(date +%F)

# 특정 키워드별 에러 로그 확인
cat reports/$(date +%F)/Error.txt
cat reports/$(date +%F)/OOMKilled.txt
```
**테스트가 끝나면, crontab을 통해 logs/와 reports/에 발생한 오류가 주기적으로 자동 기록·분류되는 것을 확인할 수 있습니다**

<br>

---


## 🛠️ 트러블슈팅

### 중복 로그 저장 문제

* 초기에 `docker logs` 사용 시, 실행할 때마다 전체 로그가 중복 수집되는 문제가 있었음.

<br>

### 해결 방법

* **컨테이너별 offset 관리**:

  * 각 컨테이너의 로그 파일 크기(바이트 단위)를 기록
  * 다음 실행 시 해당 offset 이후만 수집
  * → 중복 없는 로그 수집 가능

<br>


**✏️쉘 스크립트**

- 현재 파일 크기 기록
 ```
offset_file="$OFFSET_DIR/${fullid}.offset"

    if [ ! -f "$offset_file" ]; then
        wc -c < "$clog" > "$offset_file"
    fi

    last_offset=$(cat "$offset_file")

```

- offset 갱신
```

    wc -c < "$clog" > "$offset_file"

```

## 📝 고찰

컨테이너 환경에서는 예상치 못한 중지나 오류가 빈번하게 발생하므로,  
이를 자동으로 기록하고 분석하는 체계가 운영 안정성에 큰 도움이 됨을 알 수 있었다.  
이번 프로젝트는 Docker 수준에서의 기본적인 자동화를 실현했으며,  
앞으로 Kubernetes 환경으로 확장한다면 중앙집중식 로깅과 모니터링으로 발전시킬 수 있을 것이다.  

특히 단순한 로그 수집을 넘어서 알림, 대시보드 시각화와 연계한다면  
서비스 장애 대응 속도를 한층 더 높일 수 있을 것으로 기대된다.  



