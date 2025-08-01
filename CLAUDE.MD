# 사내 메일 서버 구축 PRD
## LDAP 하이브리드 인증 + PostgreSQL + MinIO Maildir

---

## 📋 1. 프로젝트 개요

### 1.1 목표
- **하이브리드 인증 시스템**: LDAP 구축 완료된 계열사는 LDAP 인증, 미구축 계열사는 로컬 DB 인증
- **확장 가능한 메일 시스템**: 3,000명 사용자, 50~100개 도메인 지원
- **Virtual Mailbox 방식**: 시스템 계정 없이 PostgreSQL + MinIO 기반 메일 저장
- **다중 프로토콜 지원**: SMTP, IMAP, POP3 완전 지원
- **통합 관리 시스템**: 웹 기반 관리자 인터페이스 제공

### 1.2 주요 특징
- 사용자당 기본 할당량: **1GB**
- 예상 전체 용량: **3TB** (3,000명 × 1GB)
- 동시 접속자: **500명** (피크 타임)
- 서버 스펙: Dell 760 (16Core, 32GB RAM)
- 스토리지: 별도 MinIO 서버 클러스터

---

## 🏗️ 2. 시스템 아키텍처

```
                    +-------------------------+
                    |    사용자 (클라이언트)   |
                    |   SMTP/IMAP/POP3       |
                    +-----------+-------------+
                                |
                    +-----------v-------------+
                    |      HAProxy/Nginx      | ← Load Balancer
                    +-----------+-------------+
                                |
                    +-----------v-------------+
                    |     Postfix (SMTP)      | ← 메일 수신/발신
                    +-----------+-------------+
                                |
                    +-----------v-------------+
                    |   Dovecot (IMAP/POP3)   | ← 메일박스 접근
                    +-----+----------+--------+
                          |          |
              +-----------v+    +----v---------+
              | PostgreSQL  |    |    MinIO     |
              | Virtual Map |    | (S3 Storage) |
              +------+------+    +--------------+
                     |
        +------------+-------------+
        |                          |
   +----v-----+            +------v-------+
   |   LDAP   |            | 관리자 웹페이지 |
   |(테크랩스)|            | (로컬 계정관리) |
   +----------+            +---------------+
```

---

## 🔧 3. 구성 요소별 상세 요구사항

### 3.1 하이브리드 인증 시스템

#### 3.1.1 LDAP 인증 (테크랩스 등)
- **프로토콜**: LDAPS (636 포트)
- **서버**: ldaps://ldap.company.kr
- **Base DN**: `dc=company,dc=kr`
- **인증 속성**: `uid`, `mail`, `userPassword`
- **보안**: TLS 필수, 익명 바인딩 금지
- **메타데이터**: 사용자 메일주소, 부서정보 자동 동기화

#### 3.1.2 로컬 DB 인증 (LDAP 미구축 계열사)
- **저장소**: PostgreSQL `local_users` 테이블
- **암호화**: bcrypt 해시 (cost=12)
- **관리방식**: 웹 관리자 페이지를 통한 CRUD
- **인증 우선순위**: LDAP → 로컬 DB 순서로 시도

#### 3.1.3 인증 플로우
```
사용자 로그인 시도
      ↓
LDAP 서버 인증 시도
      ↓
성공? → 로그인 완료
      ↓ (실패)
로컬 DB 인증 시도
      ↓
성공? → 로그인 완료
      ↓ (실패)
인증 실패 처리
```

### 3.2 Virtual Mailbox 시스템

#### 3.2.1 메일박스 구조
- **시스템 계정**: `vmail` (uid: 5000, gid: 5000)
- **디렉토리 구조**: `domain.com/username/Maildir/`
- **저장 위치**: MinIO S3 버킷 (`mailboxes`)
- **경로 형식**: `s3://mailboxes/domain.com/username/Maildir/`

#### 3.2.2 메일 접근 프로토콜
- **IMAP**: 포트 993 (TLS 필수), 주 사용 프로토콜
- **POP3**: 포트 995 (TLS 필수), 레거시 지원
- **POP3 설정**: 서버 보관 7일 기본, 사용자 설정 가능
- **SMTP**: 포트 587 (STARTTLS), 포트 465 (TLS)

### 3.3 PostgreSQL 데이터베이스 설계

#### 3.3.1 핵심 테이블 구조

**domains 테이블**
```sql
CREATE TABLE domains (
    id SERIAL PRIMARY KEY,
    domain_name VARCHAR(255) UNIQUE NOT NULL,
    transport VARCHAR(255) DEFAULT 'virtual:',
    company_id INTEGER,
    auth_type ENUM('ldap', 'local') DEFAULT 'local',
    ldap_base_dn VARCHAR(255),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**virtual_users 테이블**
```sql
CREATE TABLE virtual_users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255), -- 로컬 인증용 (bcrypt)
    domain_id INTEGER REFERENCES domains(id),
    maildir_path VARCHAR(500),
    quota BIGINT DEFAULT 1073741824, -- 1GB in bytes
    used_quota BIGINT DEFAULT 0,
    auth_source ENUM('ldap', 'local') DEFAULT 'local',
    is_active BOOLEAN DEFAULT TRUE,
    last_login TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**virtual_aliases 테이블**
```sql
CREATE TABLE virtual_aliases (
    id SERIAL PRIMARY KEY,
    source VARCHAR(255) NOT NULL,
    destination VARCHAR(255) NOT NULL,
    domain_id INTEGER REFERENCES domains(id),
    is_active BOOLEAN DEFAULT TRUE
);
```

#### 3.3.2 인덱스 및 성능 최적화
```sql
CREATE INDEX idx_virtual_users_email ON virtual_users(email);
CREATE INDEX idx_virtual_users_domain ON virtual_users(domain_id);
CREATE INDEX idx_virtual_users_active ON virtual_users(is_active);
CREATE INDEX idx_domains_name ON domains(domain_name);
```

### 3.4 MinIO S3 스토리지

#### 3.4.1 기본 설정
- **버킷명**: `mailboxes`
- **접근 방식**: Dovecot Object Storage Plugin
- **인증**: Access Key / Secret Key
- **엔드포인트**: 내부망 고정 IP (https://minio.company.kr)
- **지역**: `kr-central-1` (사용자 정의)

#### 3.4.2 버킷 정책 및 구조
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "MailServerAccess",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::mailserver:user/dovecot"},
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::mailboxes/*"
    }
  ]
}
```

#### 3.4.3 객체 라이프사이클
- **삭제된 메일**: 30일 후 완전 삭제
- **임시 파일**: 7일 후 자동 삭제
- **백업 복제**: 다른 리전으로 일일 복제

---

## 🛡️ 4. 보안 요구사항

### 4.1 네트워크 보안
- **암호화**: 모든 통신 TLS 1.3 이상 필수
- **방화벽**: 필요 포트만 개방 (25, 587, 993, 995, 443)
- **VPN**: 관리자 접속은 VPN 경유 필수
- **DDoS**: Fail2ban + 레이트 리미팅 적용

### 4.2 메일 보안
- **스팸 필터**: Rspamd + ClamAV 연동
- **인증 정책**: SPF, DKIM, DMARC 완전 구현
- **첨부파일**: 실행파일 차단, 100MB 제한
- **바이러스 검사**: 실시간 스캔 (ClamAV)

### 4.3 접근 제어
- **관리자 인증**: MFA (Multi-Factor Authentication) 필수
- **API 보안**: JWT 토큰 + IP 화이트리스트
- **감사 로그**: 모든 관리 작업 로깅
- **세션 관리**: 30분 비활성시 자동 로그아웃

---

## 📊 5. 성능 및 용량 계획

### 5.1 성능 요구사항
- **동시 접속자**: 500명 (피크 타임 기준)
- **메일 처리량**: 시간당 10,000건
- **응답 시간**: IMAP 로그인 < 2초, 메일 목록 < 3초
- **처리량**: MinIO 읽기 500MB/s, 쓰기 200MB/s
- **가용성**: 99.9% 업타임 보장

### 5.2 용량 계획
| 항목 | 설정값 | 비고 |
|------|--------|------|
| 사용자당 할당량 | 1GB | 기본값, 개별 조정 가능 |
| 전체 예상 용량 | 3TB | 3,000명 × 1GB |
| MinIO 클러스터 | 6TB | 복제 + 여유분 포함 |
| PostgreSQL | 100GB | 메타데이터 + 로그 |
| 백업 스토리지 | 9TB | 3-2-1 백업 원칙 |

### 5.3 확장성 계획
- **수직 확장**: CPU/RAM 업그레이드 용이성
- **수평 확장**: Dovecot Proxy + 다중 백엔드 지원
- **스토리지 확장**: MinIO 클러스터 노드 추가
- **지리적 분산**: 향후 다중 리전 지원 고려

---

## 🔧 6. 소프트웨어 스택

| 구성 요소 | 버전 | 설치 방식 | 목적 |
|-----------|------|-----------|------|
| Ubuntu Server | 22.04 LTS | ISO 설치 | 기본 OS |
| Postfix | 3.6+ | APT | SMTP 서버 |
| Dovecot | 2.3+ | APT + 플러그인 | IMAP/POP3 서버 |
| PostgreSQL | 14+ | APT | 메타데이터 DB |
| MinIO | Latest | Binary/Docker | S3 호환 스토리지 |
| Rspamd | 3.0+ | APT | 스팸/바이러스 필터 |
| ClamAV | 0.105+ | APT | 안티바이러스 |
| Nginx | 1.22+ | APT | 웹서버/프록시 |
| Django | 4.2 LTS | pip/venv | 관리자 웹페이지 |
| Redis | 7.0+ | APT | 세션/캐시 |

---

## 🖥️ 7. 관리 시스템

### 7.1 관리자 웹 인터페이스

#### 7.1.1 주요 기능
- **사용자 관리**: 로컬 계정 생성/수정/삭제/비활성화
- **도메인 관리**: 도메인 추가/삭제, 인증 방식 설정
- **할당량 관리**: 개별/그룹별 용량 조정
- **모니터링**: 실시간 시스템 상태, 메일 통계
- **로그 조회**: 인증/전송/오류 로그 검색
- **보안 관리**: 스팸/바이러스 차단 현황

#### 7.1.2 기술 스택
- **백엔드**: Django 4.2 LTS + Django REST Framework
- **프론트엔드**: Bootstrap 5 + jQuery + Chart.js
- **인증**: Django 기본 인증 + MFA
- **API**: RESTful API (JSON)

#### 7.1.3 권한 관리
```python
# 권한 레벨 정의
PERMISSION_LEVELS = {
    'SUPER_ADMIN': ['*'],  # 모든 권한
    'DOMAIN_ADMIN': ['manage_users', 'view_logs', 'manage_aliases'],
    'SUPPORT': ['view_users', 'view_logs', 'reset_password'],
    'VIEWER': ['view_users', 'view_stats']
}
```

### 7.2 자동화 스크립트
- **LDAP 동기화**: 1시간마다 사용자 정보 동기화
- **용량 체크**: 매일 할당량 초과 사용자 알림
- **로그 로테이션**: 주간 로그 아카이브
- **백업 스크립트**: 일일 자동 백업 수행

---

## 📈 8. 모니터링 및 로깅

### 8.1 모니터링 시스템

#### 8.1.1 구성 요소
- **Prometheus**: 메트릭 수집 및 저장
- **Grafana**: 대시보드 및 시각화
- **Alertmanager**: 알림 관리
- **Node Exporter**: 시스템 메트릭
- **Custom Exporter**: 메일 시스템 메트릭

#### 8.1.2 모니터링 대상
| 카테고리 | 메트릭 | 임계값 |
|----------|--------|--------|
| 시스템 | CPU, Memory, Disk | CPU > 80%, Memory > 85% |
| 네트워크 | Bandwidth, Connections | Connection > 1000 |
| 메일 | Queue Size, Processing Time | Queue > 100, Time > 5s |
| 스토리지 | MinIO Usage, IOPS | Usage > 80%, IOPS > 1000 |
| 데이터베이스 | Connection, Query Time | Connection > 100, Time > 1s |

#### 8.1.3 알림 채널
- **긴급**: SMS + 전화 (서비스 중단)
- **경고**: 메일 + Mattermost (성능 저하)
- **정보**: 메일 (일반 알림)

### 8.2 로깅 시스템

#### 8.2.1 로그 종류
```bash
# Postfix 로그
/var/log/mail.log          # 메일 전송/수신 로그
/var/log/mail.err          # 메일 오류 로그

# Dovecot 로그
/var/log/dovecot.log       # IMAP/POP3 접속 로그
/var/log/dovecot-auth.log  # 인증 로그

# 애플리케이션 로그
/var/log/mailserver/app.log    # 웹 관리자 로그
/var/log/mailserver/api.log    # API 호출 로그
/var/log/mailserver/sync.log   # LDAP 동기화 로그
```

#### 8.2.2 로그 관리
- **보존 기간**: 6개월 (압축 저장)
- **로테이션**: 일별 로테이션, 압축 보관
- **검색**: ELK Stack 또는 Grafana Loki 활용
- **백업**: 외부 스토리지로 주간 백업

---

## 🧪 9. 테스트 및 검증 계획

### 9.1 기능 테스트

| 테스트 항목 | 테스트 방법 | 성공 기준 |
|-------------|-------------|-----------|
| LDAP 인증 | LDAP 사용자로 SMTP/IMAP 로그인 | 로그인 성공, 메일 송수신 가능 |
| 로컬 인증 | 관리자 페이지에서 생성한 계정으로 로그인 | 로그인 성공, 메일 송수신 가능 |
| 하이브리드 인증 | LDAP 실패 시 로컬 DB 인증 시도 | Fallback 인증 성공 |
| Virtual Mailbox | 메일 송수신 후 MinIO 버킷 확인 | Maildir 구조로 저장 확인 |
| 다중 도메인 | 다른 도메인 간 메일 송수신 | 도메인별 독립적 처리 |
| 할당량 제한 | 1GB 초과 메일 수신 시도 | 거부 처리 및 알림 발송 |

### 9.2 성능 테스트

#### 9.2.1 부하 테스트 시나리오
```bash
# 동시 로그인 테스트
siege -c 500 -t 5m https://mail.company.kr/login

# 메일 송신 부하 테스트
for i in {1..1000}; do
  swaks --to test@company.kr --from load$i@company.kr &
done

# IMAP 접속 부하 테스트
python3 imap_load_test.py --concurrent=200 --duration=600
```

#### 9.2.2 성능 기준
- **로그인 응답시간**: < 2초 (95 percentile)
- **메일 전송 처리**: < 5초 (95 percentile)
- **IMAP 폴더 로딩**: < 3초 (95 percentile)
- **동시 접속**: 500명 이상 지원
- **시스템 리소스**: CPU < 80%, Memory < 85%

### 9.3 보안 테스트

#### 9.3.1 취약점 스캔
- **Nessus/OpenVAS**: 시스템 취약점 스캔
- **OWASP ZAP**: 웹 애플리케이션 보안 테스트
- **SSL Labs**: TLS 설정 검증
- **메일 헤더 분석**: SPF/DKIM/DMARC 검증

#### 9.3.2 침투 테스트
- **브루트포스 공격**: Fail2ban 동작 확인
- **SQL 인젝션**: 웹 인터페이스 보안 검증
- **세션 하이재킹**: 세션 관리 보안 확인
- **권한 상승**: 관리자 권한 우회 시도

---

## 💾 10. 백업 및 재해복구

### 10.1 백업 전략 (3-2-1 원칙)

#### 10.1.1 PostgreSQL 백업
```bash
# 일일 풀백업
pg_dump -h localhost -U mailserver mailserver_db > backup_$(date +%Y%m%d).sql

# 실시간 WAL 백업
postgresql.conf:
  wal_level = replica
  archive_mode = on
  archive_command = 'cp %p /backup/wal/%f'
```

#### 10.1.2 MinIO 백업
```bash
# 다른 MinIO 클러스터로 복제
mc mirror --watch minio1/mailboxes minio2/mailboxes

# 외부 S3로 일일 백업
mc mirror minio1/mailboxes aws-s3/backup-mailboxes
```

#### 10.1.3 설정 백업
```bash
# Git 기반 형상관리
/etc/postfix/     → git repository
/etc/dovecot/     → git repository
/etc/postgresql/  → git repository
```

### 10.2 재해복구 계획

#### 10.2.1 복구 목표
- **RTO (Recovery Time Objective)**: 4시간
- **RPO (Recovery Point Objective)**: 1시간
- **서비스 우선순위**: 메일 수신 → 발신 → 웹관리

#### 10.2.2 복구 절차
1. **시스템 복구** (1시간): OS 재설치 + 소프트웨어 설치
2. **데이터 복구** (2시간): PostgreSQL + MinIO 데이터 복원
3. **서비스 복구** (1시간): 설정 적용 + 서비스 시작 + 테스트

---

## 🚀 11. 구현 로드맵

### 11.1 Phase 1: 인프라 구축 (2주)
**담당**: 인프라팀, DB팀
- [ ] Ubuntu 22.04 서버 설치 및 기본 설정
- [ ] PostgreSQL 14 설치 및 데이터베이스 스키마 생성
- [ ] MinIO 클러스터 구성 및 버킷 생성
- [ ] TLS 인증서 발급 및 방화벽 설정
- [ ] 모니터링 시스템 (Prometheus + Grafana) 설치

### 11.2 Phase 2: 메일 시스템 핵심 구축 (3주)
**담당**: 백엔드팀, 보안팀
- [ ] Postfix + Dovecot 설치 및 기본 설정
- [ ] LDAP 인증 연동 개발 및 테스트
- [ ] PostgreSQL Virtual Mailbox 연동
- [ ] MinIO Object Storage Plugin 설정
- [ ] 로컬 DB 인증 시스템 개발
- [ ] 하이브리드 인증 로직 구현

### 11.3 Phase 3: 관리 시스템 개발 (3주)
**담당**: 풀스택팀
- [ ] Django 기반 관리자 웹페이지 개발
- [ ] 사용자 관리 CRUD API 개발
- [ ] 도메인 관리 기능 구현
- [ ] 모니터링 대시보드 구현
- [ ] 할당량 관리 시스템 개발
- [ ] 로그 조회 및 검색 기능

### 11.4 Phase 4: 보안 및 최적화 (2주)
**담당**: 보안팀, 성능팀
- [ ] Rspamd + ClamAV 스팸/바이러스 필터 설정
- [ ] SPF/DKIM/DMARC 정책 구현
- [ ] Fail2ban 및 레이트 리미팅 설정
- [ ] SSL/TLS 보안 강화
- [ ] 성능 튜닝 및 최적화
- [ ] 부하 테스트 및 성능 검증

### 11.5 Phase 5: 테스트 및 배포 (1주)
**담당**: QA팀, 전체팀
- [ ] 통합 테스트 수행
- [ ] 보안 취약점 스캔
- [ ] 사용자 수용 테스트 (UAT)
- [ ] 운영 매뉴얼 작성
- [ ] 배포 및 서비스 오픈

---

## 📚 12. 향후 확장 계획

### 12.1 단기 확장 (2개월 이내)
- **Webmail 연동**: Roundcube 또는 SOGo 통합
- **모바일 앱**: iOS/Android 전용 메일 앱
- **API 확장**: RESTful API 고도화
- **SSO 연동**: SAML/OAuth2 지원

### 12.2 중기 확장 (3개월 이내)
- **그룹웨어 통합**: 캘린더, 주소록, 파일 공유
- **메신저 연동**: Mattermost, Slack 알림
- **AI 기능**: 스팸 분류 학습, 메일 분류
- **다중 리전**: 지리적 분산 배포

---


**문서 버전**: v2.0  
**최종 수정일**: 2025-08-01  
**승인자**: 김형님  
**검토자**: 전체 개발팀
