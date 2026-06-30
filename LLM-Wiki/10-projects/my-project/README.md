---
type: project-index
status: active
tags:
  - ai-workflow
  - project
  - babBTI
  - meal-matching
---

# 밥BTI — 취향기반 학과 밥 약속 매칭 서비스

## 이 프로젝트는 무엇을 만드는가

학과 단톡방 사람들의 음식 취향(카테고리·매이망·예산·결정스타일)을 입력받아,
취향이 잘 맞는 **밥메이트 3명** + **추천 식당 3곳**을 자동으로 매칭해주는 웹 서비스.

- 단일 `index.html` 파일 → 어디서든 열면 바로 동작
- Firebase Realtime DB 연동 시 같은 룸 코드끼리 **실시간 그룹 매칭**

## 왜 이 프로젝트를 하는가

- AWS 부트캠프 팀1 과제: "10명의 사용자를 빠르게 만들 수 있는 서비스 제작"
- AI 워크플로우 맥락: Claude 기반 개발 + LLM-Wiki 기록 + GitHub PR 사이클

## 최종 결과물

- `index.html` — 취향 입력 → 밥BTI 타입 → 밥메이트 + 식당 추천 + 통계 + 라이브 룸
- GitHub Pages 또는 S3 배포 링크

## 현재 상태

- [x] 기본 UI + 취향 입력 폼
- [x] 유사도 알고리즘 (자카드 + 가중합)
- [x] 밥BTI 타입 판정 (9종)
- [x] 샘플 15인 데이터 + 식당 12곳
- [x] 라이브 룸 (Firebase 미설정 시 데모 모드)
- [x] 통계 막대 그래프
- [ ] Firebase 실제 연결
- [ ] GitHub Pages / S3 배포

## 에이전트가 먼저 읽을 노트

- [[decision-log]]
- [[progress]]

## 핵심 알고리즘 요약

```
similarity(a, b) =
  자카드(a.cats, b.cats) × 0.40
  + (1 - |a.spicy - b.spicy| / 5) × 0.30
  + (1 - |a.budget - b.budget| / 10000) × 0.18
  + amountSim × 0.12
```
