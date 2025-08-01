# 📋 메일 서버 구축 TODO List

## ✅ 완료된 작업 (2025-08-01)

### 인프라 기초 설정
- [x] Ubuntu 22.04 LTS 서버 기본 설정
- [x] Git 저장소 초기화 및 GitHub 연동
  - Repository: https://github.com/hardline7/tech-mail-server
  - .gitignore 설정으로 민감한 파일 보호

### PostgreSQL 설정
- [x] PostgreSQL 14 설치 및 서비스 활성화
- [x] 데이터베이스 사용자 생성
  - 사용자: techmail
  - 비밀번호: .env 파일에 저장
- [x] mailserver 데이터베이스 생성
- [x] 연결 테스트 완료

### MinIO S3 스토리지 설정
- [x] MinIO 버킷 생성: mail-tlabs-team
- [x] 정책 생성: mail-server-policy
- [x] 사용자 생성: mailserver-user
- [x] Access Key 발급 및 .env 파일에 저장
- [x] 버전 관리 활성화
- [x] 접근 권한 설정 (R/W)

### 환경 설정 파일
- [x] .env 파일 생성 및 권한 설정 (600)
- [x] PostgreSQL 연결 정보 저장
- [x] MinIO S3 설정 정보 저장
- [x] LDAP 서버 정보 저장
- [x] GitHub 토큰 저장

## 🔄 진행 중인 작업

### Phase 1: 인프라 구축 (남은 작업)

#### 1. PostgreSQL 데이터베이스 스키마 생성 🚨 우선순위: 높음
- [ ] domains 테이블 생성
  ```sql
  CREATE TABLE domains (
      id SERIAL PRIMARY KEY,
      domain_name VARCHAR(255) UNIQUE NOT NULL,
      transport VARCHAR(255) DEFAULT 'virtual:',
      company_id INTEGER,
      auth_type VARCHAR(10) DEFAULT 'local' CHECK (auth_type IN ('ldap', 'local')),
      ldap_base_dn VARCHAR(255),
      is_active BOOLEAN DEFAULT TRUE,
      created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
  );
  ```
- [ ] virtual_users 테이블 생성
- [ ] virtual_aliases 테이블 생성
- [ ] 필요한 인덱스 생성
- [ ] 테스트 데이터 입력

#### 2. TLS 인증서 설정
- [ ] Let's Encrypt 설치 (certbot)
- [ ] 도메인 확인 및 인증서 발급
- [ ] 자동 갱신 설정 (cron)
- [ ] Nginx 리버스 프록시 설정

#### 3. 방화벽 설정
- [ ] UFW 활성화
- [ ] 필요 포트 개방
  - 25 (SMTP)
  - 587 (SMTP Submission)
  - 465 (SMTPS)
  - 993 (IMAPS)
  - 995 (POP3S)
  - 443 (HTTPS)
  - 80 (HTTP - Let's Encrypt용)
- [ ] SSH 포트 보안 설정

## 📅 다음 단계 작업 (Phase 2: 메일 서버 핵심)

### 1. Postfix 설치 및 설정
- [ ] Postfix 패키지 설치
- [ ] main.cf 기본 설정
  - [ ] 호스트명 및 도메인 설정
  - [ ] 네트워크 인터페이스 설정
  - [ ] TLS 설정
- [ ] master.cf 서비스 설정
- [ ] PostgreSQL 연동 설정
  - [ ] virtual_mailbox_domains
  - [ ] virtual_mailbox_maps
  - [ ] virtual_alias_maps
- [ ] SASL 인증 설정

### 2. Dovecot 설치 및 설정
- [ ] Dovecot 코어 및 플러그인 설치
  - [ ] dovecot-core
  - [ ] dovecot-imapd
  - [ ] dovecot-pop3d
  - [ ] dovecot-pgsql
- [ ] 기본 설정 파일 구성
  - [ ] dovecot.conf
  - [ ] 10-mail.conf (메일 위치)
  - [ ] 10-auth.conf (인증 방식)
  - [ ] 10-ssl.conf (SSL/TLS)
- [ ] PostgreSQL 인증 연동
  - [ ] dovecot-sql.conf.ext 작성
  - [ ] 사용자 쿼리 설정
  - [ ] 비밀번호 쿼리 설정
- [ ] MinIO S3 플러그인 설정
  - [ ] obox 플러그인 설치
  - [ ] S3 연결 설정
  - [ ] Maildir 경로 매핑
- [ ] 하이브리드 인증 구현
  - [ ] LDAP 인증 모듈 설정
  - [ ] 로컬 DB 폴백 로직
  - [ ] 인증 스크립트 작성

### 3. 보안 설정
- [ ] Rspamd 설치 및 설정
  - [ ] 기본 설정
  - [ ] 베이지안 필터 학습
  - [ ] DKIM 서명 설정
- [ ] ClamAV 안티바이러스 설정
  - [ ] clamd 설치
  - [ ] 실시간 스캔 설정
  - [ ] Rspamd 연동
- [ ] Fail2ban 설정
  - [ ] Postfix 필터
  - [ ] Dovecot 필터
  - [ ] 차단 규칙 설정
- [ ] SPF/DKIM/DMARC 설정
  - [ ] DNS 레코드 추가
  - [ ] DKIM 키 생성
  - [ ] 정책 설정

## 🎯 우선순위 작업 (즉시 실행)

1. **PostgreSQL 스키마 생성** - 메일 시스템의 기초
2. **TLS 인증서 설정** - 보안 통신 필수
3. **방화벽 설정** - 기본 보안 구성
4. **Postfix 기본 설치** - SMTP 서버 구축 시작

## 📌 참고사항

- 모든 설정 파일은 Git으로 버전 관리
- 테스트는 항상 개발 환경에서 먼저 수행
- 각 단계별 백업 수행
- 설정 변경 시 서비스 재시작 필수
- 로그 모니터링으로 문제 조기 발견

## 🔗 관련 문서

- [CLAUDE.MD](/opt/mail/CLAUDE.MD) - 프로젝트 전체 요구사항
- [task.md](/opt/mail/task.md) - 마일스톤 기반 작업 목록
- [.env](/opt/mail/.env) - 환경 설정 (Git 제외)
- [docs/minio-setup.md](/opt/mail/docs/minio-setup.md) - MinIO 설정 상세 (Git 제외)

---

최종 업데이트: 2025-08-01