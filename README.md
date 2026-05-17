# BookTracker (MyBookManager)

> iOS 독서 관리 앱 — 도서 검색, 상태 추적, 영수증 관리, 독서 통계를 하나의 앱에서

<p align="center">
  <img src="https://img.shields.io/badge/Swift-5-F05138?logo=swift&logoColor=white" />
  <img src="https://img.shields.io/badge/SwiftUI-17.6+-007AFF?logo=swift&logoColor=white" />
  <img src="https://img.shields.io/badge/TCA-1.0+-6C3FC5" />
  <img src="https://img.shields.io/badge/Supabase-2.0+-3FCF8E?logo=supabase&logoColor=white" />
  <img src="https://img.shields.io/badge/Localization-KO%20%7C%20EN%20%7C%20JA-blue" />
</p>

<!-- 📸 앱 스크린샷을 여기에 추가하세요
<p align="center">
  <img src="docs/images/home.png" width="180" />
  <img src="docs/images/search.png" width="180" />
  <img src="docs/images/library.png" width="180" />
  <img src="docs/images/stats.png" width="180" />
</p>
-->

---

## 프로젝트 개요

BookTracker는 독서 습관을 체계적으로 관리할 수 있는 iOS 앱입니다. Google Books API를 통한 도서 검색, 읽기 상태 추적, 구매/대여 영수증 관리, 캘린더 기반 독서 통계를 제공합니다. **The Composable Architecture(TCA)** 기반으로 설계되었으며, 단순한 기능 구현을 넘어 **성능 최적화**, **타입 안전성**, **테스트 가능한 구조**에 중점을 두고 개발했습니다.

## 주요 기능

| 탭 | 기능 | 설명 |
|---|---|---|
| **홈** | 오늘의 독서 기록 | 일간/주간 독서 기록, 도서 현황 요약 |
| **검색** | 도서 검색 & 등록 | Google Books API 연동, 커스텀 도서 생성, 검색 기록 관리 |
| **라이브러리** | 내 서재 관리 | 상태별 도서 관리(읽는 중/읽고싶은/완독/중단), 컬렉션, 영수증 |
| **통계** | 독서 분석 | 독서 캘린더, 완독 도서 썸네일 캘린더, 독서 리포트 |
| **설정** | 프로필 & 계정 | 프로필 수정, 데이터 관리, 이용약관, 계정 삭제 |

**그 외:**
- Apple / Google 소셜 로그인 + 이메일 인증
- 도서별 메모, 평점(0-5), 리뷰 작성
- 구매/대여 영수증 관리 (다중 통화 지원)
- 캘린더 이미지 캡처 → 사진첩 저장
- 한국어 / English / 日本語 3개국어 지원

## 기술 스택

| 영역 | 기술 |
|---|---|
| **Language** | Swift 5 |
| **UI** | SwiftUI (iOS 17.6+) |
| **Architecture** | The Composable Architecture (TCA) |
| **Backend** | Supabase (Auth, Database, RPC) |
| **External API** | Google Books API |
| **Local Storage** | CoreData (검색 기록, 영수증, 커스텀 도서) |
| **Image** | Kingfisher (캐싱, 리트라이) |
| **Build** | Tuist + Swift Package Manager |
| **CI/CD** | Xcode Cloud |
| **Testing** | Swift Testing + TCA TestStore (38개 테스트 파일) |

## 아키텍처

```
BookTracker
├── App/                    # 앱 진입점, Root Feature
├── Features/               # Feature-based 모듈 구조
│   ├── Auth/               # 로그인/회원가입
│   └── Main/
│       ├── Home/           # 홈 (독서 기록)
│       ├── Search/         # 검색 (도서 검색, 추천, 커스텀 도서)
│       ├── Library/        # 라이브러리 (내 서재, 컬렉션, 영수증)
│       ├── Stat/           # 통계 (캘린더, 리포트)
│       └── Setting/        # 설정
├── Services/               # 비즈니스 로직 (15+ 서비스)
├── Dependencies/           # TCA Dependency Injection
└── Utils/                  # 공유 타입, 확장, 스타일
```

각 Feature는 독립적인 **`@Reducer`** 로 구성되며, `State` → `Action` → `Effect` 단방향 데이터 흐름을 따릅니다. 자식 Feature는 `Scope`, `@Presents`, `StackState` 등을 통해 부모와 합성됩니다.

## 기술적 의사결정 & 문제 해결

이 프로젝트에서 해결한 4가지 기술적 문제를 정리했습니다.

### 1. CoreData 대량 삭제 최적화

**문제:** 루프 기반 개별 삭제로 1,000건 이상 시 ~400ms 소요 (O(n) 메모리 + O(n) SQL)

**해결:** `NSBatchDeleteRequest`로 전환하여 SQLite 레벨에서 단일 쿼리 실행

| 데이터 건수 | Before | After | 개선 |
|---|---|---|---|
| 100건 | ~50ms | ~10ms | **5x** |
| 1,000건 | ~400ms | ~20ms | **20x** |

3개 서비스, 5개 함수에 적용. `mergeChanges`로 in-memory cache 동기화 처리.

> [상세 문서 →](docs/portfolio/01-coredata-batch-delete-optimization.md)

### 2. TCA Scope Enum 전환 시 Effect 누수 해결

**문제:** 수동 `Scope`에서 enum case 전환 시 이전 case의 비동기 Effect가 자동 취소되지 않아 콘솔 경고 반복 발생

**해결:** `@Reducer enum` + `@Presents` + `.ifLet` 패턴으로 마이그레이션 → case 전환 시 자동 취소

| 항목 | Before | After |
|---|---|---|
| Effect 취소 | `.concatenate(cancel, transition)` × 4곳 | 자동 |
| 내부 전용 액션 | 2개 | 0개 |
| 코드 변화 | — | **순 -64줄** |

> [상세 문서 →](docs/portfolio/02-tca-scope-enum-transition-warning.md)

### 3. Make Illegal States Unrepresentable

**문제:** 9개 Feature에서 `isLoading` + `isError` 등 Bool 플래그 조합으로 불가능한 상태(impossible states)가 타입 시스템에서 허용됨

**해결:** `LoadingState` enum (idle/loading/loaded/error) 도입으로 상태를 단일 프로퍼티로 통합

| 항목 | Before | After |
|---|---|---|
| Bool 변수 | 19개 | **9개 enum으로 통합** |
| 불가능한 상태 조합 | 최대 4가지/Feature | **0가지** |
| 적용 범위 | — | 9개 Feature, 27개 파일 |

> [상세 문서 →](docs/portfolio/03-make-illegal-states-unrepresentable.md)

### 4. ImageRenderer + AsyncImage 캡처 이슈

**문제:** `ImageRenderer`가 `AsyncImage`의 UIKit 기반 `ProgressView`를 렌더링하지 못해 캘린더 캡처 실패 + Alpha 채널로 인한 메모리 2배 사용

**해결:** 캡처 전용 뷰에서 Kingfisher 메모리 캐시 동기 로드 + `UIGraphicsImageRenderer`로 opaque 변환

- AsyncImage → Kingfisher `KFImage` (화면 표시) + `ImageCache` 동기 로드 (캡처)
- Alpha 채널 제거로 파일 크기 감소 + 디코딩 메모리 **50% 절감**

> [상세 문서 →](docs/portfolio/04-image-renderer-async-image-issue.md)

## 테스트

TCA의 `TestStore`를 활용한 Reducer 단위 테스트를 작성했습니다.

- **38개 테스트 파일** — 전체 Feature Reducer 커버리지
- 상태 전환, 비동기 Effect, 의존성 주입 모두 테스트
- `TestFixtures`로 테스트 데이터 일관성 유지

```swift
// 예시: 독서 기록 생성 테스트
@Test func doneButtonTapped_createsRecord() async {
    let store = TestStore(initialState: HomeFeature.State()) {
        HomeFeature()
    }
    store.dependencies.readingRecordService.create = { _ in .success(record) }

    await store.send(.doneButtonTapped) {
        $0.isTodayRecordUpdating = true
    }
    await store.receive(\.createTodayResponse) { ... }
}
```

## 프로젝트 구조 요약

| 항목 | 수치 |
|---|---|
| Swift 소스 파일 | 167개 |
| 테스트 파일 | 38개 |
| Feature 모듈 | 5개 메인 탭 + Auth |
| 서비스 레이어 | 15+ |
| 지원 언어 | 3개 (한/영/일) |
| 외부 의존성 | 3개 (TCA, Supabase, Kingfisher) |

## 환경 설정

### 요구 사항

- Xcode 16+
- iOS 17.6+
- [Tuist](https://tuist.io)
- [mise](https://mise.jdx.dev) (선택)

### 설정

```bash
# 1. 저장소 클론
git clone https://github.com/roddyisthebest/BookTracker.git

# 2. 환경 변수 설정
# Config/App-Debug.xcconfig, Config/App-Release.xcconfig에
# SUPABASE_URL, SUPABASE_ANON_KEY, GOOGLE_BOOKS_API_KEY 설정

# 3. Tuist로 프로젝트 생성
tuist generate

# 4. Xcode에서 빌드 & 실행
```

## 라이선스

이 프로젝트는 개인 포트폴리오 용도로 공개되었습니다.
