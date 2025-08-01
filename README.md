# 🚀 Tech Mail Server

엔터프라이즈급 하이브리드 메일 서버 시스템 구축 프로젝트

## 📋 프로젝트 개요

이 프로젝트는 3,000명 규모의 사용자와 50-100개 도메인을 지원하는 대규모 메일 서버 시스템을 구축하는 것을 목표로 합니다.

### 주요 특징

- **하이브리드 인증**: LDAP + PostgreSQL 로컬 DB 폴백
- **클라우드 스토리지**: MinIO S3 호환 오브젝트 스토리지
- **Virtual Mailbox**: 시스템 계정 없이 가상 메일박스 운영
- **다중 프로토콜**: SMTP, IMAP, POP3 완전 지원
- **웹 관리 시스템**: Django 기반 관리자 인터페이스
- **엔터프라이즈 보안**: SPF/DKIM/DMARC, 스팸/바이러스 필터링

## 🏗️ 시스템 아키텍처

```
┌─────────────────┐
│ 사용자 클라이언트 │
└────────┬────────┘
         │
┌────────▼────────┐
│  Load Balancer  │
│ (HAProxy/Nginx) │
└────────┬────────┘
         │
┌────────▼────────┐     ┌──────────────┐
│ Postfix (SMTP)  │────▶│ Rspamd/ClamAV│
└────────┬────────┘     └──────────────┘
         │
┌────────▼────────┐
│Dovecot(IMAP/POP)│
└───┬────────┬────┘
    │        │
┌───▼───┐ ┌─▼──────┐
│PostgreSQL│ │MinIO S3│
└────┬────┘ └────────┘
     │
┌────▼────┐ ┌────────────┐
│  LDAP   │ │Django Admin│
└─────────┘ └────────────┘
```

## 🔧 기술 스택

| 구성 요소 | 버전 | 용도 |
|-----------|------|------|
| Ubuntu Server | 22.04 LTS | 운영체제 |
| Postfix | 3.6+ | SMTP 서버 |
| Dovecot | 2.3+ | IMAP/POP3 서버 |
| PostgreSQL | 14+ | 사용자 데이터베이스 |
| MinIO | Latest | S3 호환 메일 스토리지 |
| Django | 4.2 LTS | 관리자 웹 인터페이스 |
| Rspamd | 3.0+ | 스팸 필터링 |
| ClamAV | 0.105+ | 바이러스 스캔 |

## 📁 프로젝트 구조

```
/opt/mail/
├── config/           # 서비스 설정 템플릿
├── scripts/          # 자동화 스크립트
├── web/              # Django 관리 시스템
├── docs/             # 프로젝트 문서
├── tests/            # 테스트 스크립트
├── .env              # 환경 변수 (Git 제외)
├── CLAUDE.MD         # 상세 PRD 문서
├── task.md           # 마일스톤 기반 작업 목록
└── todo.md           # 현재 진행 상황
```

## 🚀 빠른 시작

### 필수 요구사항

- Ubuntu 22.04 LTS
- 최소 32GB RAM, 16 Core CPU
- 공인 IP 주소
- 유효한 도메인
- MinIO 클러스터 접근 권한

### 설치 진행 상황

✅ **완료된 작업**
- PostgreSQL 14 설치 및 설정
- MinIO S3 버킷 및 정책 구성
- Git 저장소 초기화
- 환경 변수 설정 (.env)

🔄 **진행 예정**
- DNS 레코드 설정
- TLS 인증서 발급
- Postfix/Dovecot 설치
- 하이브리드 인증 구현
- Django 관리 시스템 개발

## 📊 성능 목표

- **사용자 수**: 3,000명
- **도메인 수**: 50-100개
- **동시 접속**: 500명 (피크 타임)
- **메일 처리량**: 시간당 10,000건
- **사용자 할당량**: 1GB/사용자
- **가용성**: 99.9% 업타임

## 🛡️ 보안 기능

- **암호화**: 모든 통신 TLS 1.3
- **인증**: LDAP + 로컬 DB 하이브리드
- **스팸 차단**: Rspamd + 베이지안 필터
- **바이러스 스캔**: ClamAV 실시간 검사
- **메일 인증**: SPF, DKIM, DMARC
- **접근 제어**: Fail2ban, 레이트 리미팅

## 📚 문서

- [프로젝트 요구사항 (PRD)](CLAUDE.MD)
- [작업 목록](task.md)
- [진행 상황](todo.md)
- [월-화 작업 계획](docs/week-plan-monday-tuesday.md)

## 🤝 기여 방법

1. 이 저장소를 Fork 합니다
2. 기능 브랜치를 생성합니다 (`git checkout -b feature/AmazingFeature`)
3. 변경사항을 커밋합니다 (`git commit -m 'Add some AmazingFeature'`)
4. 브랜치에 푸시합니다 (`git push origin feature/AmazingFeature`)
5. Pull Request를 생성합니다

## 📝 라이선스

이 프로젝트는 내부 사용 목적으로 개발되었습니다.

## 👥 팀

- **프로젝트 관리자**: 김형님
- **개발팀**: TechLabs 개발팀
- **인프라팀**: TechLabs 인프라팀

## 📞 연락처

프로젝트 관련 문의: admin@techlabs.co.kr

---

**Last Updated**: 2025-08-01
