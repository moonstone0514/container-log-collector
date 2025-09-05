# container-log-collector


네, 문석님 👍 지금까지 만든 흐름을 바탕으로 깔끔한 **README.md** 초안을 작성해드릴게요.
이 README는 그대로 GitHub에 올려도 되고, 나중에 다이어그램이나 예시 스크린샷만 추가하면 완성도가 올라갑니다.

---

# 📄 README.md

```markdown
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

# 매 정각 리포트 생성
0 * * * * /home/ubuntu/docker-log-project/container-log-split.sh
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

### 가짜 로그 삽입 (테스트용)

```bash
echo "[2025-09-05 21:00:00] [CONTAINER:web01] ERROR: Simulated failure" \
  >> logs/web01-error-$(date +%F).log
```

---

## 📌 참고 사항

* 로그 소스

  * 컨테이너 stdout/stderr → `/var/lib/docker/containers/<CID>/<CID>-json.log`
  * 컨테이너 이벤트 로그 → `journalctl -u docker.service`
* offset 기반으로 동작하므로, 이미 처리된 로그는 중복 기록되지 않음
* 단순 학습/테스트 목적이면 echo 로 가짜 로그를 넣어도 됩니다

