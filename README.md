# Linux_Log_Practice
Linux Log Lab은 리눅스 환경에서 웹서버(Nginx)와 시스템 자원(CPU, Memory, Disk, Network) 로그를 자동으로 수집하고 분석하는 실습용 프로젝트입니다.

🌟 프로젝트: 단일 서버 웹/CPU 로그 수집 및 분석
<details> <summary>📌 1️⃣ 프로젝트 개요</summary>
목적

단일 Nginx 서버 + 사용자 1명 환경에서 웹 트래픽과 CPU 상태를 주기적으로 수집

수집 데이터를 awk로 분석하고, 주기적으로 자동 보고서 생성

단시간 내 충분한 로그 확보 → 부하 툴 사용 가능

분석 목표

웹로그

요청 수

시간대별 트래픽

CPU 로그

CPU 사용률

Load Average

평균 CPU 사용률

서버 리소스 모니터링

CPU, Memory, Disk, Network 사용량을 5분마다 CSV로 기록

추후 Grafana/Excel 시각화 연계 가능

</details> <details> <summary>🛠 2️⃣ 시스템 서비스 관리 로그 개념</summary>
2-1. 시간 및 날짜 서비스

systemd-timedated.service

시스템 시간과 날짜 관리

정상 시작, 재로드, 완료

예시 로그:

2025-09-05T11:48:38.492393+09:00 시작

2025-09-05T11:48:38.988000+09:00 reload 완료 (489ms)

2025-09-05T11:48:39.072090+09:00 완전히 시작

2-2. snapd 관련 서비스

Snap 패키지 관리 + 보안(AppArmor) 서비스 반복 시작/중지

주요 서비스

snapd.seeded.service: snap 초기화 완료 대기

snapd.apparmor.service: AppArmor 프로필 로드 ("No profiles to load" 정상)

snapd.mounts.target: snap 패키지 마운트 관리

건너뛴 서비스

snapd.autoimport.service: 새로운 스냅 없음

snapd.core-fixup.service: 수정 필요 없음

snapd.recovery-chooser-trigger.service: 복구 모드 아님

snapd.snap-repair.timer: 수리 필요 없음

2-3. 기타 서비스

반복 시작/종료 서비스: t-news.service, apt-news.service, esm-cache.service, sysstat-collect.service, packagekit.service, fwupd.service, fwupd-refresh.service

sysstat-collect.service → 10분 간격 시스템 활동 통계 수집

fwupd 관련 경고: /dev/sr0: No medium found → 심각 오류 아님

</details> <details> <summary>⏰ 3️⃣ CRON 작업</summary>

반복 실행되는 CRON:

/home/ubuntu/scripts/moda_collect_logs.sh

/home/ubuntu/scripts/system_cron_summary.sh >> /home/ubuntu/logs/cron_log.txt 2>&1

command -v debian-sa1 > /dev/null && debian-sa1 1 1 → 시스템 활동 수집

CRON 경고:

No MTA installed, discarding output → 메일 전송 없음, 영향 없음

</details> <details> <summary>⚠️ 4️⃣ 커널/하드웨어 경고</summary>

workqueue: drm_fb_helper_damage_work hogged CPU for >10000us

그래픽 드라이버/프레임버퍼 관련 CPU 점유

GUI 사용하지 않거나 부하 낮으면 무시 가능

반복 시 CPU 점유율 증가 가능 → 모니터링 필요

</details> <details> <summary>👤 5️⃣ 사용자 세션</summary>

session-XX.scope - Session XX of User ubuntu

새로운 터미널/SSH 세션 생성 기록

정상 이벤트

</details> <details> <summary>📊 6️⃣ 서비스/CRON/경고 종합</summary>
구분	내용	영향/조치
✅ 정상 서비스	systemd-timedated, snapd, t-news/apt-news/sysstat/fwupd 등 반복 수행	없음, 정상적 시스템 관리
✅ CRON	moda_collect_logs.sh, system_cron_summary.sh, debian-sa1	출력 메일 없음 → 영향 없음
⚠️ 주의 메시지	drm_fb_helper_damage_work CPU 점유	GUI/그래픽 사용하지 않으면 무시 가능, 반복 시 모니터링
⚠️ 주의 메시지	fwupd /dev/sr0: No medium found	CD/DVD 없음, 무시 가능
⚠️ 주의 메시지	No MTA installed	CRON 메일 전송 실패, 무시 가능
</details> <details> <summary>🖥 7️⃣ 실습 흐름</summary>
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

</details> <details> <summary>🛠 8️⃣ 환경 준비</summary>
sudo apt update
sudo apt install nginx sysstat ifstat apache2-utils -y
sudo systemctl enable nginx --now

mkdir -p ~/logs/csv ~/logs/old ~/scripts


~/logs/ → 로그 저장

~/logs/csv/ → CSV 저장

~/scripts/ → 스크립트 저장

</details> <details> <summary>⚙️ 9️⃣ 사용 기술/도구</summary>
구분	도구	역할
🌐 웹 서버	Nginx	웹 트래픽 로그 자동 기록
🚦 트래픽 발생	ApacheBench(ab)	단시간 트래픽 생성
🕒 자동화	cron	5분 단위 실행
📝 로그 분석	awk	웹로그/CPU 로그 통계
🖥 CPU 모니터링	top, mpstat, uptime	CPU 상태 수집
💾 로그 관리	Linux 파일 시스템	파일 누적, timestamp 포함
📊 시각화	CSV	후속 분석
</details> <details> <summary>🚀 10️⃣ 트래픽 생성 스크립트</summary>

~/scripts/random_traffic.sh:

#!/bin/bash
URL="http://127.0.0.1/"
while true; do
    TOTAL_REQUESTS=$(shuf -i 50-200 -n 1)
    CONCURRENCY=$(shuf -i 1-10 -n 1)
    ab -n $TOTAL_REQUESTS -c $CONCURRENCY $URL > /dev/null 2>&1
    SLEEP_TIME=$(shuf -i 1-5 -n 1)
    sleep $SLEEP_TIME
done

</details> <details> <summary>📥 11️⃣ 로그 수집 스크립트</summary>

~/scripts/collect_logs.sh:

#!/bin/bash
LOG_DIR="$HOME/logs"
mkdir -p $LOG_DIR
TS=$(date +%Y%m%d_%H%M)

WEB_LOG_FILE="$LOG_DIR/web_report_$TS.txt"
NGINX_LOG="/var/log/nginx/access.log"
awk '{count++} END {print count}' $NGINX_LOG > $WEB_LOG_FILE

CPU_LOG_FILE="$LOG_DIR/cpu_report_$TS.txt"
top -bn1 | grep "Cpu(s)" > $CPU_LOG_FILE
mpstat 1 1 >> $CPU_LOG_FILE
uptime >> $CPU_LOG_FILE

</details> <details> <summary>🕹 12️⃣ 자동 실행 (cron)</summary>
crontab -e


추가:

*/1 * * * * /home/log_test/scripts/collect_logs.sh


1분마다 웹로그 + CPU + 서버 리소스 기록

</details> <details> <summary>📈 13️⃣ 로그 분석 및 모니터링</summary>

웹로그: ~/logs/web_report_YYYYMMDD_HHMM.txt

CPU 로그: ~/logs/cpu_report_YYYYMMDD_HHMM.txt

주요 분석 항목

총 요청 수

HTTP 상태 코드별 요청 수

접속 IP Top 10

사용자 에이전트 Top 5

요청 페이지 Top

시간대별 요청 수 (시/분 단위)

CPU 사용률(top, mpstat) & Load Average

</details> <details> <summary>📊 14️⃣ 실시간 모니터링 스크립트</summary>

~/scripts/monitor.sh:

#!/bin/bash
SUMMARY_DIR=~/logs/summary
while true; do
    clear
    cat $(ls -t $SUMMARY_DIR/web_accumulated_*.txt | head -1)
    cat $(ls -t $SUMMARY_DIR/web_latest_*.txt | head -1)
    cat $(ls -t $SUMMARY_DIR/cpu_summary_*.txt | head -1)
    sleep 5
done

</details> <details> <summary>📝 15️⃣ 분석 결과 요약</summary>

요청 대부분: / + /test.html

/index.html 거의 없음

트래픽 로컬 테스트용, CPU/Load 안정적

CPU 사용률:

us: 04.5%, sy: 04.8%, idle: 92~98%

Load Average 안정적

서버 정상 작동, 성능 경고 필요 없음

</details>
