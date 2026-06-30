# 밥BTI — 취향 기반 밥약 매칭

> 음식 취향 30초 입력 → 찰떡 밥메이트 + 추천 메뉴 + AI 코멘트 자동 생성  
> AWS 부트캠프 국민대 AI Workflow 팀 1

**배포 URL**: http://kmu-agent-46-babbti-s3.s3-website-us-east-1.amazonaws.com

---

## 서비스 구조

단일 `index.html` 파일 하나로 동작합니다. 서버·빌드 없이 브라우저에서 바로 열리며, Firebase 연결 시 실시간 다인 매칭이 활성화됩니다.

```
랜딩 페이지
  └── 실시간 참여자 수 뱃지 (지금 N명이 매칭 중)

로그인 (닉네임 + 비번)
  ├── 재방문 자동 로그인 / 계정 전환
  └── 로그아웃 버튼

활성 룸 목록 (실시간) → 바로 입장 가능

취향 입력 — 3단계 위저드
  ├── 1단계  음식 선택  카테고리 칩(9종) → 세부 메뉴(35종) 토글
  ├── 2단계  맛 설정   매운맛(0~5) · 짠맛(0~3) · 단맛(0~3) · 향신료 선호
  └── 3단계  스타일    예산 슬라이더 · 결정 방식(내가/같이/아무거나)

결과 — 5탭 네비게이션
  ├── 내 밥BTI   9종 타입 판정 카드 + 🔗 결과 공유 버튼
  ├── 밥메이트   7차원 유사도 TOP 3 (실제참여 뱃지 / 샘플 뱃지 구분)
  │             + AI 매칭 코멘트 (Bedrock) · 실시간 참여자 반영
  ├── 추천 메뉴  그룹 취향 기반 TOP 3 → 📍 GPS 주변 실제 식당
  ├── 라이브 룸  실시간 조 편성 · 채팅 · 📋 룸 코드 복사
  └── 취향 통계  매운맛·짠맛·단맛·향신료·카테고리 분포 차트
```

---

## 실행 방법

### 로컬 (Firebase 없이 단일 기기 데모)

```bash
python3 -m http.server 3000
# → http://localhost:3000/index.html
```

> `file://` 직접 열기는 Firebase CORS 차단 및 GPS 미동작 가능성 있음

### 배포 (S3 정적 호스팅)

```
S3 버킷 → 퍼블릭 액세스 허용 → 정적 웹 호스팅 활성화 → index.html 업로드
업데이트: 동일 파일명으로 덮어쓰기 업로드 → 즉시 반영
```

---

## AI 코멘트 — Amazon Bedrock

밥메이트 매칭 후 이 그룹이 왜 잘 맞는지 AI가 자연어로 설명합니다. **API 키 설정 불필요.**

```
브라우저 → API Gateway → Lambda → Amazon Bedrock (Claude 3 Haiku)
```

| 항목 | 값 |
|---|---|
| 모델 | `anthropic.claude-3-haiku-20240307-v1:0` |
| Lambda | `babbti-ai-comment` (Python 3.12, us-east-1) |
| API Gateway | `POST .../ai-comment` |
| IAM | `Nxt-Lambda-Bedrock-Role` — 브라우저에 자격증명 노출 없음 |

---

## 알고리즘

### 밥메이트 유사도 (7차원 가중합)

```
similarity(a, b) =
  Jaccard(cats)   × 0.32   // 음식 카테고리 겹침
  + 매운맛 근접도 × 0.18
  + 짠맛 근접도   × 0.12
  + 단맛 근접도   × 0.10
  + 향신료 일치   × 0.08
  + 예산 근접도   × 0.12
  + 결정스타일    × 0.08
```

> 콜드스타트 상황에서 협업 필터링 없이 즉시 동작하는 콘텐츠 기반 매칭

### 그룹 추천 메뉴 (평균 취향 기반)

```
menu_score = 카테고리 투표합 × 0.38 + 예산 × 0.20 + 매운맛 × 0.15
           + 짠맛 × 0.12 + 단맛 × 0.08 + 향신료 × 0.07
```

### GPS 식당 매칭 (OpenStreetMap Overpass API)

- OSM `cuisine` 태그 → 밥BTI 9개 카테고리 자동 매핑
- 거리(haversine) × 0.45 + 취향 점수 × 0.45
- 실패 시 큐레이션 목록으로 폴백 (화면 깨짐 없음)

---

## Firebase 구현

| 기능 | 방식 |
|---|---|
| 브라우저 종료 시 자동 퇴장 | `onDisconnect().remove()` |
| 좀비 참여자 제거 | 10분 초과 타임스탬프 클라이언트 필터 |
| 참여자 중복 방지 | `set(encodeURIComponent(nick))` — 재제출 시 동일 키 덮어쓰기 |
| 룸 멤버 중복 방지 | `roomRef.child(encodeURIComponent(nick)).set()` — 재입장 시 덮어쓰기 |
| 연결 상태 표시 | `.info/connected` 리스너 |
| 좋아요 동시 집계 | `transaction()` 원자적 처리 |
| 실시간 참여자 동기화 | `on("value")` 리스너 + `numChildren()` 카운트 |
| 로그인 / 재방문 | Firebase DB 닉네임+비번 + localStorage 캐시 |
| 실시간 채팅 | `/rooms/{code}/chat` + `limitToLast(50)` |
| XSS 방지 | `esc()` 이스케이프 전처리 |

---

## 팀 역할

| 담당 | 내용 |
|---|---|
| ChoHyeonChan | Firebase 백엔드 / 실시간 동기화 / 라이브 카운터 / 채팅 / Bedrock AI / S3 배포 / UI 고급화 / 참여자·룸 중복 방지 |
| junseok0929 | 로그아웃 / 좋아요 실시간 / 1:1 DM 모달 / 모바일 반응형 |
| juneddoha | GPS 식당 추천 / 3단계 위저드 / 5탭 결과 UI / 칩 선택 UX |
