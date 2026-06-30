# 밥BTI — 취향 기반 학과 밥 약속 매칭

> 음식 취향 30초 입력 → 찰떡 밥메이트 + 식당 자동 추천  
> AWS 부트캠프 국민대 AI Workflow 팀 1

## 서비스 개요

단일 `index.html` 파일 하나로 동작하는 취향 기반 밥 약속 매칭 서비스입니다.  
서버·빌드 환경 없이 브라우저에서 바로 열리며, Firebase Realtime Database 연결 시 실시간 그룹 매칭이 활성화됩니다.

```
index.html
 ├── 로그인          닉네임 + 비번으로 계정 생성 / 자동 로그인 (localStorage)
 ├── 활성 룸 목록    현재 참여 중인 룸 실시간 표시
 ├── 취향 입력       카테고리·매운맛·짠맛·단맛·향신료·예산·결정스타일
 ├── 밥BTI 타입      9종 중 1개 판정
 ├── 밥메이트 TOP 3  7차원 유사도 알고리즘으로 매칭
 ├── 추천 식당 TOP 3 그룹 평균 취향 기반 콘텐츠 매칭
 │     └── GPS 버튼  OpenStreetMap으로 반경 800m 실제 식당 검색
 ├── 라이브 룸       Firebase 연결 시 실시간 3~4인 그룹핑
 └── 취향 통계       매운맛·짠맛·단맛·향신료·인기 카테고리 분포
```

## 실행 방법

### 로컬 데모 (Firebase 없이)
```bash
# 방법 1: Python 서버
python3 -m http.server 3000
# http://localhost:3000/index.html

# 방법 2: VS Code Live Server 확장 설치 후 index.html 우클릭 → Open with Live Server
```

> `file://` 직접 열기는 Firebase CORS 차단 및 GPS 미동작 가능성 있음

### 배포 (실서비스)

- **GitHub Pages** (권장): `Settings → Pages → main branch / (root)` → 링크 공유
- **S3 + CloudFront**: 버킷 정적 호스팅 + CloudFront HTTPS 배포
- GPS 기능은 **HTTPS 환경 필수** — `file://` 및 HTTP S3 기본 엔드포인트 불가

## Firebase 연결 (백엔드 담당자)

1. [Firebase 콘솔](https://console.firebase.google.com) → 프로젝트 만들기
2. **Realtime Database** → 데이터베이스 만들기 → **테스트 모드로 시작**
3. 프로젝트 설정 → 내 앱 → 웹 앱 등록 → `firebaseConfig` 복사
4. `index.html` 상단 `firebaseConfig` 블록에 실제 값 붙여넣기  
   - `databaseURL` 반드시 포함 (없으면 Realtime DB 연결 안 됨)
5. 탭 2개로 같은 룸 코드 입력 → 라이브 룸 "2명 참여 중" 확인

> `apiKey`는 공개돼도 괜찮지만, **발표 후 DB Rules 잠금 또는 프로젝트 삭제 필수**

## 알고리즘

### 밥메이트 유사도 (7차원 가중합)

```
similarity(a, b) =
  Jaccard(cats)    × 0.32   // 음식 카테고리 겹침
  + 매운맛 근접도  × 0.18   // |spicy_a - spicy_b| / 5
  + 짠맛 근접도    × 0.12   // |salty_a - salty_b| / 3
  + 단맛 근접도    × 0.10   // |sweet_a - sweet_b| / 3
  + 향신료 일치    × 0.08   // 같으면 1.0, 인접 0.5, 반대 0.0
  + 예산 근접도    × 0.12   // |budget_a - budget_b| / 10000
  + 결정스타일     × 0.08   // |amount_a - amount_b| / 2
```

### 식당 추천 (그룹 평균 기반)

```
resto_score =
  카테고리 투표합  × 0.38
  + 예산 근접도    × 0.20
  + 매운맛 근접도  × 0.15
  + 짠맛 근접도    × 0.12
  + 단맛 근접도    × 0.08
  + 향신료 선호    × 0.07
```

> 알고리즘 선택 이유: 취향을 1회만 입력하는 콜드스타트 상황 → 상호작용 이력 없이도 즉시 동작하는 콘텐츠 기반 방식 채택 (ALS/SVD++ 협업 필터링 제외)

### GPS 실제 식당 추천 (Overpass API)

```
실제_식당_score =
  그룹 취향 점수  × 0.45
  + 거리 점수     × 0.45   // haversine 거리, 가까울수록 높음
  + 기본 점수     × 0.10
```

- 데이터: [OpenStreetMap Overpass API](https://overpass-api.de) — API 키 불필요
- OSM `cuisine` 태그 → 밥BTI 9개 카테고리 자동 매핑
- 실패 시 기존 가상 큐레이션으로 폴백 (화면 깨짐 없음)

## Firebase 구현 패턴

| 기능 | 구현 방식 |
|---|---|
| 브라우저 종료 시 자동 퇴장 | `onDisconnect().remove()` |
| 좀비 참여자 필터 | 클라이언트 10분 타임스탬프 필터 |
| 실시간 연결 상태 표시 | `.info/connected` 리스너 |
| 좋아요 동시 집계 | `transaction()` 원자적 처리 |
| XSS 방지 | `esc()` 이스케이프 (닉네임 등 공개 입력값) |
| 참여자 실시간 동기화 | `on("value")` 리스너 (once → on 교체) |
| 로그인 / 재방문 | Firebase DB 닉네임+비번 저장 + localStorage 자동 로그인 |

## 팀 역할

| 역할 | 담당 |
|---|---|
| 프론트 UI / 취향 알고리즘 | junseok0929 |
| Firebase 백엔드 / 실시간 동기화 | ChoHyeonChan |
| GPS 실제 식당 추천 | juneddoha |
