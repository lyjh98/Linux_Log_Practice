# 🌟 프로젝트: 단일 서버 웹/CPU 로그 수집 및 분석

| 이제현 | 전수민 |
|:---:|:---:|
| [lyjh98](https://github.com/lyjh98) | [Jsumin07](https://github.com/Jsumin07) |
| <img src="https://avatars.githubusercontent.com/lyjh98" width="150px;"> | <img src="https://avatars.githubusercontent.com/Jsumin07" width="150px;"> |



## 1️⃣ 주제 기획

### 목적

- 단일 Nginx 서버 + 사용자 1명 환경에서 **웹 트래픽과 CPU 상태를 주기적으로 수집**
- 수집 데이터를 **awk**를 이용해 분석하고, 주기적으로 자동 보고서 생성
- 단시간 내 충분한 로그 확보 → 부하 툴 사용 가능

### 분석 목표

1. **웹로그**
    - 요청 수
    - 시간대별 트래픽
2. **CPU 로그**
    - CPU 사용률
    - Load Average
    - 평균 CPU 사용률

3. **서버 리소스 모니터링**

- CPU, Memory, Disk, Network 사용량을 5분마다 수집해 CSV로 기록
- → 추후 Grafana/Excel 시각화 연계 가능


## 1. 시스템 서비스 관리 (systemd & snapd) 로그 개념

### 1-1. 시간 및 날짜 서비스

- **systemd-timedated.service**
    - 시스템 시간과 날짜를 관리하는 서비스
    - 정상적으로 시작, 재로드, 완료됨
    - 로그 예시:
        - `2025-09-05T11:48:38.492393+09:00` Time & Date Service 시작
        - `2025-09-05T11:48:38.988000+09:00` reload 완료 (489ms)
        - `2025-09-05T11:48:39.072090+09:00` 서비스 완전히 시작

### 1-2. snapd 관련 서비스

- Snap 패키지 관리 및 보안(AppArmor) 관련 서비스 반복 시작/중지
- **주요 서비스**
    - `snapd.seeded.service`: snap 초기화 완료 대기
    - `snapd.apparmor.service`: AppArmor 프로필 로드
        - `"No profiles to load"` → 오류 아님, 정상
    - `snapd.mounts.target`: snap 패키지 마운트 관리
- **건너뛴 서비스** (조건 미충족)
    - `snapd.autoimport.service`: 새로운 스냅 없음
    - `snapd.core-fixup.service`: 수정 필요 없음
    - `snapd.recovery-chooser-trigger.service`: 복구 모드 아님
    - `snapd.snap-repair.timer`: 수리 필요 없음

### 1-3. 기타 서비스

- 반복 시작/종료되는 서비스:
    - `t-news.service`, `apt-news.service`, `esm-cache.service`, `sysstat-collect.service`, `packagekit.service`, `fwupd.service`, `fwupd-refresh.service`
    - 대부분 정상적으로 `start → 수행 → Deactivated successfully`
- `sysstat-collect.service` → 10분 간격 시스템 활동 통계 수집
- `fwupd` 관련 경고:
    - `/dev/sr0: No medium found` → CD/DVD 미디어 없음, 심각 오류 아님


## 2. CRON 작업

- 반복 실행되는 CRON
    - `/home/ubuntu/scripts/moda_collect_logs.sh`
    - `/home/ubuntu/scripts/system_cron_summary.sh >> /home/ubuntu/logs/cron_log.txt 2>&1`
    - `command -v debian-sa1 > /dev/null && debian-sa1 1 1` → 시스템 활동 수집
- CRON 로그 경고
    - `No MTA installed, discarding output`
        
        → 메일 전송 프로그램 없음, 출력 버려짐, 정상 동작에 영향 없음
        


## 3. 커널/하드웨어 관련 경고

- `workqueue: drm_fb_helper_damage_work hogged CPU for >10000us`
    - 그래픽 드라이버 또는 프레임버퍼 관련 작업이 CPU 점유
    - 서버 GUI 사용하지 않거나 부하 낮으면 무시 가능
    - 반복 빈도가 높으면 CPU 점유율 증가 문제 가능


## 4. 사용자 세션 관련

- `session-XX.scope - Session XX of User ubuntu`
    - 새로운 터미널/SSH 연결 세션 생성 기록
    - 정상 이벤트


## 5. 개념 종합 정리

| 구분 | 내용 | 영향/조치 |
| --- | --- | --- |
| 정상 서비스 | systemd-timedated, snapd 서비스, t-news/apt-news/sysstat/fwupd 등 반복 수행 | 없음, 정상적 시스템 관리 활동 |
| CRON | moda_collect_logs.sh, system_cron_summary.sh, debian-sa1 | 출력 메일 없음 → 영향 없음 |
| 주의 메시지 | drm_fb_helper_damage_work CPU 점유 | GUI/그래픽 사용하지 않으면 무시 가능, 반복 시 모니터링 |
| 주의 메시지 | fwupd /dev/sr0: No medium found | CD/DVD 없음, 무시 가능 |
| 주의 메시지 | No MTA installed | CRON 메일 전송 실패, 무시 가능 |

![image.png](attachment:3d993562-7a3a-4eea-b244-4f7c3cb86184:image.png)

전체 로그는 시스템 부팅, 서비스 재시작, CRON 작업 수행 과정에서 정상적으로 발생하는 이벤트가 대부분이며, 심각한 오류는 없음. 주의 메시지는 하드웨어/메일 환경 관련으로, 일반 서버 운영에는 큰 영향이 없음.

### 실습 흐름 요약

```
[트래픽 발생] → Nginx access.log 기록
           ↓
[cron 5분 단위] → collect_logs.sh 실행
           ↓
웹로그 → web_report_YYYYMMDD_HHMM.txt
CPU 로그 → cpu_report_YYYYMMDD_HHMM.txt
           ↓
awk 분석 → 요청 수, URL Top5, 상태코드, CPU 사용률 등
           ↓
보고서 / CSV / 시각화 / 알림
           ↓
오래된 로그 삭제 → 자동 관리

```

## 1️⃣ 환경 준비

```bash
sudo apt update
sudo apt install nginx sysstat ifstat apache2-utils -y
sudo systemctl enable nginx --now
```

### 디렉토리 구조

```bash
mkdir -p ~/logs/csv ~/logs/old ~/scripts
```

- `~/logs/` → 로그 저장
- `~/logs/csv/` → CSV 저장
- `~/scripts/` → 스크립트 저장

## 2️⃣ 사용 기술 및 도구

| 구분 | 도구 | 역할 |
| --- | --- | --- |
| 웹 서버 | Nginx | 웹 트래픽 로그 자동 기록 |
| 트래픽 발생 | ApacheBench(ab) | 단시간 트래픽 생성, 로그 확보 |
| 로그 수집/자동화 | cron | 5분 단위 실행 |
| 로그 분석 | awk | 웹로그/CPU 로그 데이터 처리, 통계 |
| CPU 모니터링 | top, mpstat, uptime | CPU 상태 수집 |
| 로그 관리 | Linux 파일 시스템 | 파일 누적, timestamp 포함 |
| 시각화 | CSV | 후속 분석 |


### 불규칙 랜덤 트래픽

`~/scripts/random_traffic.sh`

```bash
#!/bin/bash
URL="http://127.0.0.1/"

while true; do
    # 50~200 사이 랜덤 총 요청 수
    TOTAL_REQUESTS=$(shuf -i 50-200 -n 1)

    # 1~10 사이 랜덤 동시 요청
    CONCURRENCY=$(shuf -i 1-10 -n 1)
    echo "Sending $TOTAL_REQUESTS requests with concurrency $CONCURRENCY to $URL"

    # ApacheBench 실행
    ab -n $TOTAL_REQUESTS -c $CONCURRENCY $URL > /dev/null 2>&1

    # 1~5초 랜덤 대기 
    SLEEP_TIME=$(shuf -i 1-5 -n 1)
    echo "Sleeping for $SLEEP_MINUTES minutes ($SLEEP_TIME seconds)..."
    sleep $SLEEP_TIME
done

```

```bash
chmod +x ~/scripts/random_traffic.sh
```

실행:

```bash
~/scripts/random_traffic.sh
```

(Ctrl+C로 중지)


## 3️⃣ 로그 수집 스크립트


### (1) 통합 로그 수집

`~/scripts/collect_logs.sh`

```bash
#!/bin/bash

LOG_DIR="$HOME/logs"
mkdir -p $LOG_DIR
TS=$(date +%Y%m%d_%H%M)

# 1) 웹로그 수집
WEB_LOG_FILE="$LOG_DIR/web_report_$TS.txt"
NGINX_LOG="/var/log/nginx/access.log"

{
  echo "===== Web Log Report: $(date '+%Y-%m-%d %H:%M') ====="
  echo -e "\n1) 총 요청 수"
  awk '{count++} END {print count}' $NGINX_LOG
  echo -e "\n3) HTTP 상태 코드 분포"
  awk '{print $9}' $NGINX_LOG | sort | uniq -c | sort -nr
  echo -e "\n4) 시간대별 요청 수"
  awk -F: '{print $2":"$3}' $NGINX_LOG | sort | uniq -c | sort -nr | head -5
} > $WEB_LOG_FILE

# 2) CPU 로그 수집
CPU_LOG_FILE="$LOG_DIR/cpu_report_$TS.txt"
{
  echo "===== CPU Report: $(date '+%Y-%m-%d %H:%M') ====="
  echo -e "\n1) CPU 사용률 (top 1회 스냅샷)"
  top -bn1 | grep "Cpu(s)"
  echo -e "\n2) CPU 평균 사용률 (mpstat)"
  if command -v mpstat &> /dev/null; then
      mpstat 1 1
  else
      echo "mpstat not installed"
  fi
  echo -e "\n3) Load Average"
  uptime
} > $CPU_LOG_FILE

echo "Logs collected: $WEB_LOG_FILE , $CPU_LOG_FILE"

```

```bash
chmod +x ~/scripts/collect_logs.sh
```


## 4️⃣ 자동 실행 (crontab)

```bash
crontab -e
```

추가:

```bash
*/1 * * * * /home/log_test/scripts/collect_logs.sh
```

→ 1분마다 웹로그 + CPU + 서버 리소스 기록


## 5️⃣ 로그 확인 & 분석

### 생성 위치

- 웹로그: `~/logs/web_report_YYYYMMDD_HHMM.txt`
- CPU 로그: `~/logs/cpu_report_YYYYMMDD_HHMM.txt`

### 확인 방법

```bash
ls -l ~/logs
tail -n 20 ~/logs/web_report_*.txt
head -n 20 ~/logs/cpu_report_*.txt
```


단일 서버 웹/CPU 로그 수집 및 분석

트래픽 처리 스크립트

```jsx
#!/bin/bash

# -----------------------------
# 설정
# -----------------------------
LOG_DIR="$HOME/logs"
WEB_URLS=("http://localhost/" "http://localhost/test.html")
TRAFFIC_COUNT=10      # 테스트 트래픽 반복 횟수
TRAFFIC_SLEEP=1       # 요청 간 대기 시간(초)

# -----------------------------
# 로그 디렉토리 생성
# -----------------------------
mkdir -p "$LOG_DIR"

# -----------------------------
# 1) 테스트 트래픽 생성
# -----------------------------
for url in "${WEB_URLS[@]}"; do
  for i in $(seq 1 $TRAFFIC_COUNT); do
    curl -s "$url" > /dev/null
    sleep $TRAFFIC_SLEEP
  done
done

# -----------------------------
# 2) 웹로그 수집
# -----------------------------
WEB_LOG_FILE="$LOG_DIR/web_report_$(date +'%Y%m%d_%H%M').txt"
sudo cp /var/log/nginx/access.log "$WEB_LOG_FILE"
echo "Web log collected: $WEB_LOG_FILE"

# -----------------------------
# 3) CPU 로그 수집
# -----------------------------
CPU_LOG_FILE="$LOG_DIR/cpu_report_$(date +'%Y%m%d_%H%M').txt"
{
  echo "===== CPU Report: $(date +'%Y-%m-%d %H:%M') ====="
  echo
  echo "1) CPU 사용률 (top)"
  top -bn1 | grep "%Cpu" 
  echo
  echo "2) CPU 평균 사용률 (mpstat)"
  mpstat
  echo
  echo "3) Load Average"
  uptime
} > "$CPU_LOG_FILE"
echo "CPU log collected: $CPU_LOG_FILE"

# -----------------------------
# 4) 최신 웹로그 Top URL 분석
# -----------------------------
LATEST_WEBLOG=$(ls -t "$LOG_DIR"/web_report_*.txt | head -1)
echo
echo "===== Latest Web Log Top URLs ====="
awk '{print $7}' "$LATEST_WEBLOG" | sort | uniq -c | sort -nr

```

웹 로그 분석 모니터링 스크립트

```jsx
#!/bin/bash

SUMMARY_DIR=~/logs/summary

echo "===== 실시간 모니터링 시작 (Ctrl+C 종료) ====="
echo

while true; do
    # 최신 웹로그/CPU 집계 파일
    LATEST_WEB_ACC=$(ls -t $SUMMARY_DIR/web_accumulated_*.txt 2>/dev/null | head -1)
    LATEST_WEB_TOP=$(ls -t $SUMMARY_DIR/web_latest_*.txt 2>/dev/null | head -1)
    LATEST_CPU_SUM=$(ls -t $SUMMARY_DIR/cpu_summary_*.txt 2>/dev/null | head -1)

    # 화면 클리어
    clear

    # 출력
    if [[ -f "$LATEST_WEB_ACC" ]]; then
        echo "===== 최신 누적 URL 집계: $(basename $LATEST_WEB_ACC) ====="
        cat "$LATEST_WEB_ACC"
        echo
    else
        echo "웹 누적 로그 파일 없음."
    fi

    if [[ -f "$LATEST_WEB_TOP" ]]; then
        echo "===== 최신 5분 Top URL: $(basename $LATEST_WEB_TOP) ====="
        cat "$LATEST_WEB_TOP"
        echo
    else
        echo "웹 Top 로그 파일 없음."
    fi

    if [[ -f "$LATEST_CPU_SUM" ]]; then
        echo "===== 최신 CPU 요약: $(basename $LATEST_CPU_SUM) ====="
        cat "$LATEST_CPU_SUM"
        echo
    else
        echo "CPU 로그 파일 없음."
    fi

    # 5초마다 갱신
    sleep 5
done

```

시간 별 로그 접근 현황과 CPU 분석 결과

![image.png](attachment:9acc7148-4209-411d-bff4-5c148fb6d9ae:image.png)

표로 정리한 내용

```jsx
| 시간    | URL 최다 접속   | 요청 수 | CPU 사용률(top, sy%) | CPU 평균(mpstat, %sys) | Load Average (1분) |
| ----- | ----------- | ---- | ----------------- | -------------------- | ----------------- |
| 11:50 | /           | 59   | 3.5               | 1.12                 | 0.05              |
| 11:50 | /test.html  | 28   | 3.5               | 1.12                 | 0.05              |
| 11:50 | /index.html | 3    | 3.5               | 1.12                 | 0.05              |
| 11:55 | /           | 63   | 4.1               | 1.35                 | 0.08              |
| 11:55 | /test.html  | 30   | 4.1               | 1.35                 | 0.08              |
| 11:55 | /index.html | 4    | 4.1               | 1.35                 | 0.08              |
| 12:00 | /           | 68   | 4.3               | 1.48                 | 0.12              |
| 12:00 | /test.html  | 32   | 4.3               | 1.48                 | 0.12              |
| 12:00 | /index.html | 5    | 4.3               | 1.48                 | 0.12              |
| 12:05 | /           | 70   | 4.6               | 1.52                 | 0.10              |
| 12:05 | /test.html  | 33   | 4.6               | 1.52                 | 0.10              |
| 12:05 | /index.html | 6    | 4.6               | 1.52                 | 0.10              |
| 12:10 | /           | 72   | 4.8               | 1.60                 | 0.15              |
| 12:10 | /test.html  | 35   | 4.8               | 1.60                 | 0.15              |
| 12:10 | /index.html | 7    | 4.8               | 1.60                 | 0.15              |
| 12:15 | /           | 70   | 4.9               | 1.62                 | 0.12              |
| 12:15 | /test.html  | 34   | 4.9               | 1.62                 | 0.12              |
| 12:15 | /index.html | 6    | 4.9               | 1.62                 | 0.12              |
| 12:20 | /           | 69   | 5.0               | 1.65                 | 0.14              |
| 12:20 | /test.html  | 33   | 5.0               | 1.65                 | 0.14              |
| 12:20 | /index.html | 5    | 5.0               | 1.65                 | 0.14              |
| 12:25 | /           | 71   | 4.7               | 1.58                 | 0.11              |
| 12:25 | /test.html  | 34   | 4.7               | 1.58                 | 0.11              |
| 12:25 | /index.html | 4    | 4.7               | 1.58                 | 0.11              |
| 12:30 | /           | 72   | 4.6               | 1.55                 | 0.09              |
| 12:30 | /test.html  | 35   | 4.6               | 1.55                 | 0.09              |
| 12:30 | /index.html | 5    | 4.6               | 1.55                 | 0.09              |
| 12:35 | /           | 73   | 4.9               | 1.60                 | 0.13              |
| 12:35 | /test.html  | 36   | 4.9               | 1.60                 | 0.13              |
| 12:35 | /index.html | 6    | 4.9               | 1.60                 | 0.13              |
| 12:40 | /           | 71   | 4.5               | 1.63                 | 0.16              |
| 12:40 | /test.html  | 34   | 4.5               | 1.63                 | 0.16              |
| 12:40 | /index.html | 2    | 4.5               | 1.63                 | 0.16              |
| 12:45 | /           | 71   | 4.8               | 1.57                 | 0.01              |
| 12:45 | /test.html  | 34   | 4.8               | 1.57                 | 0.01              |
| 12:45 | /index.html | 2    | 4.8               | 1.57                 | 0.01              |

```

![image.png](attachment:f355e0d2-73ec-4015-acc7-bcfbc6dd3040:image.png)

- 파란 선: 시간별 총 요청 수
- 주황 선: CPU %sys
- 빨간 선: Load Average 1분

### 1️⃣ 웹 트래픽 현황

**누적 URL 집계 (`web_accumulated`)**

| URL | 누적 요청 수 |
| --- | --- |
| `/` | 71 |
| `/test.html` | 34 |
| `/index.html` | 2 |

**최근 5분 Top URL (`web_latest`)**

| URL | 요청 수 |
| --- | --- |
| `/` | 71 |
| `/test.html` | 34 |
| `/index.html` | 2 |
- 대부분 요청이 루트(`/`)와 `/test.html`에 집중.
- `/index.html`은 거의 사용되지 않음.
- 로그 확인 시, 요청은 로컬 호스트(::1)에서 `curl`로 연속적으로 들어오고 있음.
- 즉, 외부 트래픽보다는 내부 테스트용 요청으로 보임.


### 2️⃣ CPU 사용률 요약

**Load Average**

- 시스템 여유 상태 매우 안정적

**CPU 사용률(top 기준)**

- 유저(us) 사용률: 0~4.5%
- 시스템(sy) 사용률: 0~4.8%
- 아이들(id) 비율: 92~98%
- I/O 대기(wa): 0~3.7%
- 전반적으로 서버가 유휴 상태

**CPU 평균 사용률(mpstat 기준)**

- user/system/iowait 모두 낮음
- idle 94~100%
- spike 없이 안정적


### 3️⃣ 로그 패턴

- `/` 요청 → 연속 71회
- `/test.html` → 연속 34회
- 요청 간격 1초 단위로 반복
- `/index.html`은 극소수
- 로컬 `curl` 테스트 성격이 짙음 → 실제 부하 없음


![image.png](attachment:24138b2d-1f24-4548-94fa-4209a0ebd2de:image.png)

![image.png](attachment:e22b746f-d47c-401d-8fa1-0647879cb27f:image.png)

![image.png](attachment:b7873c27-59dc-4b19-a99e-44af9fa7f943:image.png)

### 4️⃣ 분석 내용

### 1. 총 요청 수

파일의 전체 라인 수를 세어 총 요청 수를 파악합니다.

- **명령어**:Bash
    
    `awk 'END {print "총 요청 수: " NR}' access_processed.csv`
    
- **분석 결과**:
    
    `총 요청 수: 12866`
    


### 2. HTTP 상태 코드별 요청 수

웹사이트의 성공(200), 리디렉션(304), 오류(404) 등 상태 코드별로 요청 수를 확인합니다.

- **명령어**:Bash
    
    `awk -F"," 'NR > 1 {count[$6]++} END {for (code in count) print code, count[code]}' access_processed.csv`
    
- **분석 결과**:
    
    `200 12853
    304 11
    404 1`
    


### 3. 가장 많이 접속한 IP 주소

트래픽의 주요 원천인 IP 주소를 파악합니다.

- **명령어**:Bash
    
    `awk -F"," 'NR > 1 {print $1}' access_processed.csv | sort | uniq -c | sort -nr | head -10`
    
- **분석 결과**:
    
    `12850 127.0.0.1
       13 10.0.2.2
        2 ::1`
    


### 4. 가장 많이 사용된 사용자 에이전트

주로 어떤 브라우저나 봇이 접속하는지 확인합니다.

- **명령어**:Bash
    
    `awk -F"," 'NR > 1 {print $8}' access_processed.csv | sort | uniq -c | sort -nr | head -5`
    
- **분석 결과**:
    
    `12850 ApacheBench/2.3
       13 "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML
        2 curl/8.5.0`
    


### 5. 가장 많이 요청된 페이지

사용자들이 가장 많이 방문한 페이지를 식별합니다.

- **명령어**:Bash
    
    `awk -F"," 'NR > 1 {print $4}' access_processed.csv | sort | uniq -c | sort -nr | head -5`
    
- **분석 결과**:
    
    `12864 /
        1 /favicon.ico`
    


### 6. 시간대별 접속 횟수 (시 단위)

하루 중 트래픽이 가장 많은 시간대를 파악합니다.

- **명령어**:Bash
    
    `awk -F"," 'NR > 1 {print substr($2, 13, 2)}' access_processed.csv | sort | uniq -c | sort -nr`
    
- **분석 결과**:
    
    `11856 12
     1009 11`
    


### 7. 시간대별 접속 횟수 (분 단위)

더 세밀하게 트래픽이 집중된 분 단위를 파악합니다.

- **명령어**:Bash
    
    `awk -F"," 'NR > 1 {print substr($2, 13, 5)}' access_processed.csv | sort | uniq -c | sort -nr`
    
- **분석 결과**:
    
    `2418 12:34
    2122 12:33
    1050 12:12
    1050 12:10
    1000 12:08
     950 12:07
     913 12:32
     850 12:11
     850 12:09
     500 11:57
     500 11:46
     300 12:06
     177 12:28
     100 12:13
      70 12:30
       6 12:35
       5 11:43
       2 11:31
       1 11:44
       1 11:30`
    


### 4️⃣ 결론

- 서버 정상 작동 중이며 CPU, Load 모두 안정적
- 현재 웹 트래픽은 테스트용 로컬 요청에 한정
- 성능 모니터링이나 경고 필요 없음
- 만약 외부 트래픽 증가나 부하 테스트 계획 시 `/`와 `/test.html` 집중 점검 필요
