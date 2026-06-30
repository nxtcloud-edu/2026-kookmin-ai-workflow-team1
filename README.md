# 🍚 밥BTI — 취향기반 학과 밥 약속 매칭

> 음식 취향 30초 입력 → 찰떡 밥메이트 + 식당 자동 추천  
> AWS 부트캠프 국민대 AI Workflow 팀 1

## 서비스 구조

단일 `index.html` 파일 하나로 동작합니다. 서버 없이 브라우저에서 바로 열면 됩니다.  
실시간 룸 매칭 기능은 **Firebase Realtime Database**를 붙여야 활성화됩니다.

```
index.html
 ├── 취향 입력 폼  (카테고리·매이망·예산·결정스타일)
 ├── 밥BTI 타입 결과
 ├── 밥메이트 TOP 3  (유사도 알고리즘)
 ├── 추천 식당 TOP 3  (그룹 취향 합산)
 │     └── 📍 내 주변 실제 식당 찾기  ← GPS + OpenStreetMap 으로 실제 식당 검색
 ├── 🔴 라이브 룸     ← Firebase 연결 시 실시간 활성화
 └── 📊 취향 통계
```

## Firebase 연결 방법 (백엔드 담당자)

1. [Firebase 콘솔](https://console.firebase.google.com) → 프로젝트 만들기
2. **Realtime Database** → 데이터베이스 만들기 → **테스트 모드로 시작**
3. 프로젝트 설정 → 내 앱 → 웹(`</>`) 앱 등록 → `firebaseConfig` 복사
4. `index.html` 하단 `firebaseConfig` 블록의 `PASTE_*` 값 4줄 교체
   - `apiKey`, `authDomain`, `databaseURL`, `projectId`, `appId`
   - **`databaseURL`은 반드시 포함** (없으면 Realtime DB 연결 안 됨)
5. 테스트: 브라우저 탭 2개로 같은 룸 코드 입력 → 라이브 룸에서 "2명 참여 중" 확인

> ⚠️ `apiKey`는 공개돼도 괜찮지만, **데모 끝나면 프로젝트 삭제 또는 DB 규칙 잠금**

## 배포

- **GitHub Pages** (권장): `Settings → Pages → main branch / (root)` → 링크 공유
- **S3 정적 호스팅**: 버킷 정책 퍼블릭 읽기 허용 후 `index.html` 업로드

## 알고리즘 요약

```js
similarity(a, b) =
  자카드(cats) × 0.40
  + (1 - |spicy_a - spicy_b| / 5) × 0.30
  + (1 - |budget_a - budget_b| / 10000) × 0.18
  + (결정스타일 보완) × 0.12
```

## 📍 내 주변 실제 식당 찾기 (GPS 기반)

결과 화면의 **"📍 내 주변 실제 식당 찾기"** 버튼을 누르면, 가상 큐레이션 대신
**현재 GPS 위치 주변의 진짜 식당**을 검색해서 우리 그룹 취향으로 추천해 줍니다.

- **데이터 소스**: [OpenStreetMap Overpass API](https://overpass-api.de) — **API 키 불필요**, 별도 가입/설정 없이 바로 동작
- **흐름**: `navigator.geolocation`으로 현재 좌표 → Overpass로 반경 800m 내 음식점(`restaurant`/`fast_food`/`cafe`) 검색 → OSM `cuisine` 태그를 밥BTI 9개 카테고리로 매핑 → **그룹 취향 점수(0.45) + 거리 점수(0.45) + 기본(0.10)** 으로 랭킹 → TOP 4 카드(거리 + 카카오맵 링크) 표시
- **폴백**: 위치 권한 거부 / 검색 실패 / 주변 데이터 없음 → 기존 가상 큐레이션 추천을 그대로 유지 (안 깨짐)
- ⚠️ GPS는 **HTTPS 환경**에서만 동작합니다. GitHub Pages·S3(https)·VS Code Live Server(localhost)는 OK, `file://` 직접 열기는 막힐 수 있어요.

## 팀 역할

| 역할 | 담당 |
|---|---|
| 백엔드 (Firebase 연결·배포) | — |
| 프론트 UI / 알고리즘 | — |
| 결과 카드·통계 시각화 | — |
