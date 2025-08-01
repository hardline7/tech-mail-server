# 📅 월~화 메일 서버 구축 작업 계획 (공인 IP/도메인 보유 기준)

## 🗓️ 월요일 작업 목록

### 1. DNS 레코드 설정 [🔴 중요도: 매우 높음]
**시간: 09:00-10:00**
- [ ] A 레코드: mail.domain.com → 공인 IP
- [ ] MX 레코드: domain.com → mail.domain.com (우선순위 10)
- [ ] SPF 레코드: "v=spf1 mx ip4:공인IP ~all"
- [ ] PTR 레코드 (역방향 DNS) 요청
- [ ] DNS 전파 확인 (dig, nslookup)

### 2. 방화벽 및 네트워크 설정 [🔴 중요도: 매우 높음]
**시간: 10:00-11:00**
- [ ] UFW 방화벽 규칙 설정
  ```bash
  sudo ufw allow 22/tcp    # SSH
  sudo ufw allow 25/tcp    # SMTP
  sudo ufw allow 587/tcp   # Submission
  sudo ufw allow 465/tcp   # SMTPS
  sudo ufw allow 993/tcp   # IMAPS
  sudo ufw allow 995/tcp   # POP3S
  sudo ufw allow 80/tcp    # HTTP
  sudo ufw allow 443/tcp   # HTTPS
  sudo ufw enable
  ```
- [ ] 네트워크 인터페이스 설정 확인
- [ ] 공인 IP 바인딩 확인

### 3. TLS/SSL 인증서 발급 [🔴 중요도: 매우 높음]
**시간: 11:00-12:00**
- [ ] Certbot 설치
  ```bash
  sudo apt install certbot python3-certbot-nginx
  ```
- [ ] Nginx 임시 설정 (도메인 확인용)
- [ ] Let's Encrypt 인증서 발급
  ```bash
  sudo certbot certonly --standalone -d mail.domain.com -d webmail.domain.com
  ```
- [ ] 자동 갱신 크론잡 설정
- [ ] 인증서 권한 설정

### 4. PostgreSQL 스키마 생성 [🔴 중요도: 매우 높음]
**시간: 13:00-14:30**
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
- [ ] 인덱스 생성
- [ ] 실제 도메인 및 테스트 사용자 등록

### 5. Postfix 설치 및 기본 설정 [🔴 중요도: 매우 높음]
**시간: 14:30-17:00**
- [ ] Postfix 및 관련 패키지 설치
  ```bash
  sudo apt install postfix postfix-pgsql sasl2-bin libsasl2-modules
  ```
- [ ] main.cf 설정
  - [ ] 호스트명, 도메인 설정
  - [ ] TLS 설정 (인증서 경로)
  - [ ] SASL 인증 설정
  - [ ] 메일 크기 제한 (100MB)
- [ ] master.cf 서비스 설정
- [ ] PostgreSQL 맵 파일 생성
- [ ] 기본 전송 테스트

### 6. Dovecot 기본 설치 [🟡 중요도: 높음]
**시간: 17:00-18:00**
- [ ] Dovecot 패키지 설치
  ```bash
  sudo apt install dovecot-core dovecot-imapd dovecot-pop3d dovecot-pgsql
  ```
- [ ] 기본 설정 백업
- [ ] SSL 인증서 경로 설정
- [ ] 로그 설정

## 🗓️ 화요일 작업 목록

### 7. Dovecot 상세 설정 [🔴 중요도: 매우 높음]
**시간: 09:00-11:00**
- [ ] PostgreSQL 인증 연동
  - [ ] dovecot-sql.conf.ext 설정
  - [ ] 사용자/비밀번호 쿼리
- [ ] 메일 저장 위치 설정 (임시 로컬)
- [ ] IMAP/POP3 프로토콜 설정
- [ ] 보안 설정 (SSL 필수)
- [ ] 인증 테스트

### 8. MinIO S3 연동 설정 [🟡 중요도: 높음]
**시간: 11:00-12:30**
- [ ] Dovecot obox 플러그인 설치/설정
- [ ] S3 연결 설정 (엔드포인트, 인증)
- [ ] 메일 저장 경로 S3로 변경
- [ ] 연결 테스트 및 메일 저장 확인

### 9. DKIM 설정 [🔴 중요도: 매우 높음]
**시간: 13:30-14:30**
- [ ] OpenDKIM 설치
  ```bash
  sudo apt install opendkim opendkim-tools
  ```
- [ ] DKIM 키 생성 (2048bit)
- [ ] Postfix 연동 설정
- [ ] DNS에 DKIM 레코드 추가
- [ ] DKIM 서명 테스트

### 10. 스팸/바이러스 필터 설정 [🟡 중요도: 높음]
**시간: 14:30-16:00**
- [ ] Rspamd 설치 및 기본 설정
  ```bash
  sudo apt install rspamd redis-server
  ```
- [ ] ClamAV 설치 및 설정
  ```bash
  sudo apt install clamav clamav-daemon
  ```
- [ ] Rspamd-ClamAV 연동
- [ ] 기본 스팸 규칙 설정
- [ ] 테스트 메일로 필터 확인

### 11. Fail2ban 보안 설정 [🟡 중요도: 높음]
**시간: 16:00-17:00**
- [ ] Fail2ban 설치
  ```bash
  sudo apt install fail2ban
  ```
- [ ] Postfix 필터 설정
- [ ] Dovecot 필터 설정
- [ ] 차단 규칙 설정 (5회 실패 시 30분 차단)
- [ ] 로그 모니터링

### 12. 최종 통합 테스트 [🔴 중요도: 매우 높음]
**시간: 17:00-18:00**
- [ ] 외부에서 메일 수신 테스트
- [ ] 외부로 메일 발신 테스트
- [ ] IMAP/POP3 클라이언트 접속 테스트
- [ ] SPF/DKIM/DMARC 검증
- [ ] mail-tester.com으로 점수 확인

## 🔍 작업 우선순위 요약

### 🔴 매우 높음 (필수)
1. DNS 설정 - 모든 것의 시작
2. 방화벽 설정 - 보안 기본
3. TLS 인증서 - 암호화 통신 필수
4. PostgreSQL 스키마 - 데이터 저장 기반
5. Postfix 설정 - 메일 송수신 핵심
6. Dovecot PostgreSQL 연동 - 사용자 인증
7. DKIM 설정 - 메일 신뢰도
8. 통합 테스트 - 전체 검증

### 🟡 높음 (중요)
- Dovecot 기본 설치
- MinIO S3 연동
- 스팸/바이러스 필터
- Fail2ban 설정

## 📌 주요 체크포인트

### 월요일
- **오전**: DNS 설정 완료 및 전파 확인
- **점심**: TLS 인증서 발급 완료
- **오후**: Postfix 기본 메일 송수신 가능
- **저녁**: telnet mail.domain.com 25 접속 가능

### 화요일
- **오전**: IMAP 클라이언트 접속 가능
- **점심**: MinIO에 메일 저장 확인
- **오후**: DKIM 서명 동작 확인
- **저녁**: mail-tester.com 점수 8/10 이상

## 🚨 문제 발생 시 대응

1. **DNS 전파 지연**: 로컬 hosts 파일로 임시 테스트
2. **인증서 발급 실패**: DNS 설정 재확인, 방화벽 80번 포트 확인
3. **메일 전송 실패**: 로그 확인 (/var/log/mail.log)
4. **인증 실패**: PostgreSQL 쿼리 직접 실행으로 확인

## 📞 지원 연락처
- DNS 관리자: (DNS 업체 연락처)
- 네트워크 관리자: (ISP 연락처)
- 보안 담당자: (내부 보안팀)

---
작성일: 2025-08-01
작성자: Mail Server Team