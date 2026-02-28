# 소비 리포트 : 내 소비는 평균일까? 💳

> 나의 소비 패턴을 동일 연령대 · 성별 그룹과 비교해보는 웹 서비스

---

## 📌 프로젝트 개요

본 서비스는 수백만 건의 카드 소비 데이터를 기반으로,  
로그인한 사용자의 소비 패턴을 동일 연령대 및 성별 그룹(또래 집단)과 비교 분석합니다.

사용자의 실제 소비 데이터와 또래 평균 소비 데이터를 시각적으로 비교하여  
자신의 소비 성향을 직관적으로 파악할 수 있도록 설계되었습니다.

---

## ✨ 주요 기능

### 🔐 로그인

- 고객번호(SEQ) + 비밀번호 기반 사용자 인증
- JSESSIONID 쿠키 기반 세션 관리
- **Tomcat 클러스터링 기반 세션 동기화 (2대 서버 구성)**
- `CharacterEncodingFilter`  
  → 전역 UTF-8 인코딩 처리
- `AuthFilter`  
  → 미인증 요청 차단 및 로그인 페이지로 리다이렉트

---

## 📊 메인 페이지 (소비 분석 대시보드)

로그인한 사용자의 소비 데이터를 또래 집단과 비교 분석합니다.

### 1️⃣ 개인 소비 분석
- 사용자의 소비 데이터 기준 **지출 상위 Top 3 카테고리** 추출
- 원형 차트(Pie Chart)로 시각화

### 2️⃣ 또래 소비 분석
- 동일 연령대 + 성별 기준 그룹 분류
- 해당 그룹의 평균 소비 금액 기준 **Top 3 카테고리** 추출
- 원형 차트(Pie Chart)로 시각화

### 3️⃣ 소비 패턴 비교
- 개인 소비 Top 3 vs 또래 평균 Top 3 비교
- 소비 성향 차이 직관적으로 확인 가능

---

### 🔓 로그아웃

- 세션 무효화 (invalidate)
- `SESSION_INFO` 테이블에서 세션 정보 삭제
- 완전한 로그아웃 처리

---

## 🛠 기술 스택

| 분류 | 기술 |
|------|------|
| Backend | Java, Servlet, JSP |
| WAS | Apache Tomcat 9 × 2대 (8080, 8090) |
| DB | MySQL 8 (Docker), HikariCP Connection Pool |
| Proxy / LB | Nginx (리버스 프록시 + 부하분산) |
| 인증 | HttpSession (JSESSIONID) + DB 세션 공유 |
| 개발 도구 | Eclipse IDE, MySQL |

---

## 🏗 시스템 아키텍처

```
클라이언트 (웹 브라우저 / API 클라이언트)
        │ HTTP 요청
        ▼
┌─────────────────────┐
│       Nginx         │  ← Presentation 계층 (정적 리소스, 트래픽 제어)
│  (SPOF 이슈 존재)   │    향후 Active-Standby 이중화 예정
└──────────┬──────────┘
     부하분산 (ip_hash)
    ┌───────┴────────┐
    ▼                ▼
┌────────┐      ┌────────┐
│Tomcat 1│      │Tomcat 2│  ← Application 계층 (Servlets & JSP, HikariCP)
│ :8080  │      │ :8090  │
└────┬───┘      └───┬────┘
     │   Read/Write │
     ▼              ▼
┌──────────────────────┐
│  MySQL Source (R/W)  │  ← Data 계층 (Docker Container)
└──────────────────────┘
         │ Replication
         ▼
┌──────────────────────┐
│  MySQL Replica (R)   │
└──────────────────────┘
```

---

## 🔑 세션 공유 전략

2대의 WAS 환경에서 발생할 수 있는 세션 불일치(Session Mismatch) 문제를 해결하기 위해  
**Tomcat Clustering (all-to-all) 기반 세션 복제 방식**을 적용했습니다.

| 방식 | 설명 |
|------|------|
| Tomcat Clustering (all-to-all) | 각 WAS 노드 간 세션을 메모리 기반으로 복제하여 세션 일관성 유지 |

---

### 📌 적용 효과

- 로드밸런싱 환경에서도 로그인 상태 유지
- 별도의 외부 세션 저장소 없이 구성 가능
- 2대 WAS 규모에 적합한 구조

---

## 🗄 DB 설계

| 테이블 | 설명 |
|--------|------|
| `EDU_DATA_F_2` | 전처리된 소비 이력 데이터 (중분류 제거, 대분류 기준) |
| `USER_INFO` | 사용자 정보 (고객번호, 비밀번호, 성별, 연령대) |
| `AVERAGE_DATA_F` | 연령대 + 성별 기준 평균 소비 통계 |
<img width="50%" alt="ERD_dark" src="https://github.com/user-attachments/assets/027f7890-0c5b-4f06-8332-cdf91bd8100a" />

---

## 🚀 실행 방법

환경 구축

| 순서 | 계층 | 가이드 문서 |
|------|------|------------|
| 1️⃣ | Data | [`database/setting_guide.md`](database/setting_guide.md) |
| 2️⃣ | WAS | [`tomcat/setting_guide.md`](tomcat/setting_guide.md) |
| 3️⃣ | Web | [`nginx/setting_guide.md`](nginx/setting_guide.md) |

```bash
# 1. MySQL Docker 컨테이너 실행
docker start [mysql-container]

# 2. Tomcat 1 (8080) 실행

# 3. Tomcat 2 (8090) 실행

# 4. Nginx 실행
sudo brew services start nginx

# 5. 브라우저에서 접속
http://localhost
```

---

## ☠️ 데이터 모델링 이슈: 시계열 데이터 처리에 따른 외래키 설정 제약

1. **현황**: USER_INFO는 SEQ를 PK로 가져가기 위해 AGE를 단일값(MAX)으로 가공.

2. **충돌**: EDU_DATA_F_2는 연도별 이력을 관리하므로 동일 SEQ에 대해 다수의 AGE 존재.

3. **결정**: (SEQ, AGE) 복합키 기반의 외래키 설정 시 참조 무결성 오류가 발생하여 FK 제약 조건 제외.

4. **한계**: DB 수준의 참조 무결성을 보장할 수 없으며, 상위 테이블 수정 시 하위 테이블의 데이터 정합성을 수동으로 관리해야 하는 관리적 부담 발생.
