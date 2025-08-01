# ✅ 전체 작업 목록 (Milestone 기반)

## 🚩 Milestone 1: 인프라 및 기본 환경 구축

| 작업 항목 | 담당 | 비고 |
|-----------|------|------|
| Ubuntu 22.04 LTS 설치 및 네트워크 설정 | 인프라팀 | 각 노드별 고정 IP |
| Hostname, DNS, NTP 설정 | 인프라팀 | 시간 동기화 중요 |
| TLS 인증서 발급 및 적용 (Let's Encrypt 또는 내부 CA) | 보안팀 | SMTP, IMAP, 웹 포함 |
| PostgreSQL 14 설치 및 튜닝 | DB팀 | `postgresql.conf` 최적화 |
| DB 스키마 생성 (domains, virtual_users 등) | DB팀 | PRD 기준 스키마 |
| MinIO 클러스터 구성 | 인프라팀 | 로드밸런서 포함 |
| MinIO 버킷 생성 및 정책 적용 | 인프라팀 | 버킷: `mailboxes` |
| 로컬 리포지터리 git 초기화 (/etc/postfix 등) | 운영팀 | 설정 추적용 |
| 방화벽 및 보안 정책 적용 | 보안팀 | 25, 587, 993, 995, 443 등 |

---

## 🚩 Milestone 2: 메일 서버 핵심 구성

| 작업 항목 | 담당 | 비고 |
|-----------|------|------|
| Postfix 설치 및 기본 설정 | 백엔드팀 | SMTP 설정 |
| Dovecot 설치 및 S3 Object Storage 연동 | 백엔드팀 | obox plugin 포함 |
| Dovecot + PostgreSQL 연동 설정 | 백엔드팀 | `dovecot-sql.conf.ext` 작성 |
| Postfix + PostgreSQL 연동 설정 | 백엔드팀 | virtual_maps 설정 |
| Dovecot + MinIO 연동 테스트 (Maildir 생성) | QA팀 | 샘플 계정으로 확인 |
| 하이브리드 인증 로직 구현 (LDAP → DB fallback) | 백엔드팀 | dovecot auth script |
| SPF/DKIM/DMARC 정책 적용 | 보안팀 | DNS 설정 포함 |
| Rspamd + ClamAV 설치 및 연동 | 보안팀 | Redis 연동 포함 |
| Fail2ban 설정 | 보안팀 | SMTP, IMAP 기준 |

---

## 🚩 Milestone 3: 관리자 웹 인터페이스 개발

| 작업 항목 | 담당 | 비고 |
|-----------|------|------|
| Django 프로젝트 초기화 (4.2 LTS) | 풀스택팀 | REST API 포함 |
| 사용자 CRUD 기능 구현 (로컬 계정용) | 풀스택팀 | bcrypt 기반 |
| 도메인 관리 UI 및 API 구현 | 풀스택팀 | auth_type 설정 가능 |
| 할당량 관리 기능 개발 | 풀스택팀 | quota 조회/조정 |
| 로그인 MFA 구현 (OTP 기반) | 풀스택팀 | django-otp 사용 가능 |
| 로그/모니터링 조회 기능 | 풀스택팀 | system log + api log |
| 권한 기반 관리 구현 (PERMISSION_LEVELS) | 풀스택팀 | RBAC 구조 적용 |
| RESTful API 문서화 | 풀스택팀 | Swagger/OpenAPI 기준 |

---

## 🚩 Milestone 4: 모니터링, 로깅, 백업

| 작업 항목 | 담당 | 비고 |
|-----------|------|------|
| Prometheus + Node Exporter 설치 | 운영팀 | 각 노드별 에이전트 |
| Dovecot/Postfix Exporter 구성 | 운영팀 | 큐 사이즈 등 수집 |
| Grafana 대시보드 구성 | 운영팀 | 사용자/도메인 기준 |
| Alertmanager 설정 (SMS, Mattermost) | 보안팀 | 알림 구간 정의 |
| Loki 또는 ELK Stack 로그 수집 구성 | 보안팀 | /var/log/* 포함 |
| PostgreSQL 백업 스크립트 작성 (WAL 포함) | DB팀 | 일일 자동화 |
| MinIO 백업 스크립트 구성 (mirror + 외부 S3) | 인프라팀 | `mc` 이용 |
| 설정 파일 Git 형상관리 적용 | 운영팀 | cron 또는 pre-commit hook |

---

## 🚩 Milestone 5: 테스트, 배포, 운영

| 작업 항목 | 담당 | 비고 |
|-----------|------|------|
| 통합 테스트 수행 (기능/보안/성능) | QA팀 | PRD 9.1~9.3 기준 |
| 성능 테스트 도구 실행 (siege, swaks 등) | QA팀 | 피크 타임 시나리오 |
| 보안 취약점 스캔 (Nessus, ZAP) | 보안팀 | TLS, API 취약점 등 |
| 운영 매뉴얼 작성 | 전체팀 | 설정 복원 포함 |
| 최종 배포 및 사용자 수용 테스트 (UAT) | 전체팀 | 도메인별 |
| 운영 모니터링 개시 및 알림 확인 | 운영팀 | 실시간 확인 |
