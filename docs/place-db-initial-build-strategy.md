# Place DB Initial Build Strategy

확인일: 2026-06-10

## 목적

Vibe Dating의 핵심 자산은 단순한 장소 목록이 아니라, 커플이 데이트 장소를 고를 때 필요한 맥락이 붙은 자체 Place DB다.

우리가 만들려는 DB의 핵심 질문은 다음이다.

```text
이 장소가 어떤 커플에게, 어떤 상황에서, 어떤 무드로 좋은가?
```

따라서 외부 지도 API의 장소 데이터를 그대로 저장하는 방식이 아니라, 공개/오픈 데이터와 자체 큐레이션, 사용자 피드백을 합쳐 우리만의 데이트 맥락 데이터를 쌓는 방향으로 간다.

## 핵심 결론

- 네이버/카카오/구글 지도 API를 우리 DB의 원본처럼 장기 저장하는 것은 약관 리스크가 있다.
- 사업자로 등록된 모든 장소를 가져올 수 있는 공개 전수 DB는 사실상 없다.
- 국세청 사업자등록정보 API는 사업자번호를 알고 있을 때 상태/진위 확인을 하는 용도이며, 사업자 목록을 전수 제공하는 API가 아니다.
- 공방, 원데이클래스, 스튜디오, 팝업 같은 장소는 공공데이터에서 누락될 수 있다.
- 초기 DB는 공공/오픈 데이터 seed + 지도 API 보강 + 자체 검수 + 사용자/업주 등록으로 구축하는 것이 현실적이다.

권장 방향:

```text
공공/오픈 데이터로 넓게 깔기
+ 카카오/네이버/구글로 장소 매칭과 현재성 보정
+ LLM/룰 기반으로 1차 태깅
+ 사람이 데이트 적합 후보를 검수
+ 사용자 호/불호와 업주 등록으로 long-tail 보완
```

## 주요 데이터 소스

### 1. 소상공인 상가정보

공식 데이터:

- https://www.data.go.kr/data/15083033/fileData.do
- https://www.data.go.kr/data/15012005/openapi.do

용도:

- 음식점, 카페, 술집, 디저트, 소매, 서비스, 교육, 여가 업종의 기본 seed.
- 영업 중인 전국 상가업소 데이터.
- 상호명, 업종코드, 업종명, 지번주소, 도로명주소, 경도, 위도 등을 제공.

장점:

- 전국 단위로 넓다.
- 비용 무료.
- 이용허락범위 제한 없음으로 표시되어 있다.
- 서울 핫플 권역 후보를 대량으로 뽑기 좋다.

한계:

- 데이트 적합도, 분위기, 리뷰, 평점, 사진, 영업시간은 없다.
- 업종 분류가 데이트 맥락과 바로 맞지 않는다.
- 공방/체험형 장소는 업종이 흩어져 있을 수 있다.

수집 전략:

```text
서울 전체 적재
-> 핫플 권역 필터
-> 음식/카페/주점뿐 아니라 교육/여가/소매/개인서비스까지 넓게 포함
-> 데이트 부적합 업종 제거
-> 지도 API로 존재 여부와 상세 URL 보강
```

### 2. 지방행정 인허가 데이터

공식 사이트:

- https://www.localdata.go.kr/
- https://www.localdata.go.kr/portal/portalDataGuide.do?menuNo=30002
- https://www.localdata.go.kr/devcenter/apiGuide.do?menuNo=20002

용도:

- 인허가가 필요한 업종의 운영 상태, 주소, 좌표, 영업상태 보강.
- 음식점, 공연, 관광, 문화기획, 영화, 음악, 체육, 미용, 전문교육기관, 식품, 숙박 등.

장점:

- 전체 자료 다운로드와 변동분 API 구조가 있어 운영형 DB에 맞다.
- 영업/정상, 휴업, 폐업, 취소/말소/정지 등 상태를 볼 수 있다.
- 좌표와 주소를 제공한다.
- 특정 업종의 개폐업 추적에 좋다.

한계:

- 모든 사업자를 포함하지 않는다. 인허가/신고 대상 업종 중심이다.
- 공방이 별도 인허가 업종이 아닌 경우 누락될 수 있다.
- 좌표계가 중부원점TM(EPSG:5174)인 경우가 있어 좌표계 변환이 필요할 수 있다.

수집 전략:

```text
전체 데이터 다운로드로 초기 적재
-> 서울/핫플 권역 필터
-> 식품/문화/생활/전문교육기관 중심으로 후보 추출
-> 변동분 API로 신규/폐업/변경 반영
```

### 3. 한국관광공사 TourAPI

공식 데이터:

- https://www.data.go.kr/data/15101578/openapi.do

용도:

- 관광지, 문화시설, 행사, 축제, 여행코스, 레포츠, 쇼핑, 음식점 계열 보강.

장점:

- 이미지, 소개, 상세 정보 계열 데이터를 활용할 수 있다.
- 관광/문화/행사성 데이트 후보에 강하다.
- 저작권에 구애 없이 활용 가능한 정보를 선별해 제공한다고 안내되어 있다.

한계:

- 일반 카페/식당/작은 공방 커버리지는 약할 수 있다.
- 최신 핫플 반영은 지도 API나 수동 큐레이션이 필요하다.

수집 전략:

```text
서울 지역 기반 수집
-> 관광지/문화시설/행사/쇼핑/음식점 분리
-> 이미지와 소개 정보 보강
-> 행사 데이터는 시작일/종료일 기준으로 만료 관리
```

### 4. 서울 열린데이터광장

공식 데이터:

- 서울시 문화공간 정보: https://data.seoul.go.kr/dataList/OA-15487/A/1/datasetView.do
- 서울시 문화행사 정보: https://data.seoul.go.kr/dataList/OA-15486/S/1/datasetView.do

용도:

- 서울 MVP의 문화공간, 공연, 전시, 행사 후보 보강.

장점:

- 서울 MVP와 직접 맞다.
- 문화행사 정보는 분류, 자치구, 공연/행사명, 장소, 날짜, 이용요금, 위치, 시작일, 종료일 등을 제공한다.
- 매일 갱신되는 데이터가 있어 현재성 있는 행사 후보를 만들 수 있다.

한계:

- 음식점/카페/술집/공방은 커버하지 못한다.
- 행사는 만료 관리가 필수다.

### 5. 카카오 로컬 API

관련 문서:

- [kakao-local-api-spec-and-usage.md](api/kakao-local-api-spec-and-usage.md)

용도:

- 권역별 장소 후보 보강.
- 장소명 + 주소 매칭.
- 카카오 place id, place url, 카테고리, 좌표 보강.
- 공방/원데이클래스 같은 long-tail 키워드 탐색.

장점:

- 네이버보다 반경/사각형/페이지네이션 검색이 좋다.
- 카테고리 검색과 키워드 검색을 모두 제공한다.
- 카카오 place id가 있어 중복 제거에 유용하다.

한계:

- 평점, 리뷰 수, 사진, 메뉴, 영업시간은 제공하지 않는다.
- 한 검색 조합에서 노출 가능한 결과 수가 제한된다.
- 카카오 응답 데이터를 장기 저장/가공/광고 모델과 결합하려면 약관 검토가 필요하다.

수집 전략:

```text
권역 중심 좌표/radius 기반 category search
+ 권역별 keyword search
+ grid/tile 분할
+ kakao place id 기준 dedupe
+ 자체 canonical place와 연결
```

### 6. 네이버/구글 API

관련 문서:

- [naver-local-api-and-place-data-strategy.md](api/naver-local-api-and-place-data-strategy.md)
- [google-maps-api-research.md](api/google-maps-api-research.md)

용도:

- 네이버: 상세 링크/현재성 확인/사용자 외부 이동 보조.
- 구글: rating, review count, price level 등 보조 신호 확인.

주의:

- 둘 다 우리 DB의 원본으로 쓰기보다는 보강/검증/연결 용도로 시작한다.
- 원본 데이터 장기 저장, 리뷰/사진/메뉴 저장, 광고 결합은 약관 검토가 필요하다.

## 공방/원데이클래스 누락 대응

공방류는 사업자등록상 업종이 하나로 모이지 않는다.

가능한 업종 분산:

- 교육서비스
- 공예품 소매
- 제조업
- 문화/예술 서비스
- 취미/여가 서비스
- 기타 개인 서비스
- 통신판매/예약 기반 서비스

따라서 `업종=공방`만 찾는 방식은 위험하다.

대응 전략:

```text
1. 소상공인 상가정보에서 교육/여가/소매/개인서비스까지 넓게 포함
2. LOCALDATA에서 문화/생활/전문교육기관/체육 관련 업종 보강
3. 카카오 로컬 API로 권역별 공방 키워드 검색
4. 사용자/업주 등록 플로우 운영
5. 사람이 공방/체험 후보를 우선 검수
```

권장 키워드:

- 공방
- 원데이클래스
- 도예공방
- 가죽공방
- 향수공방
- 캔들공방
- 반지공방
- 베이킹클래스
- 플라워클래스
- 유리공방
- 목공방
- 미술공방
- 드로잉카페
- 터프팅
- 라탄공방
- 비누공방
- 가드닝클래스
- 소품샵
- 사진관
- 셀프스튜디오

검색 조합:

```text
{권역명} + {키워드}
중심좌표 + radius + {키워드}
rect + {키워드}
```

예:

```text
성수 향수공방
연남 원데이클래스
한남 플라워클래스
을지로 유리공방
홍대 반지공방
```

## 초기 MVP 범위

권역:

- 성수
- 연남/홍대
- 한남/이태원

후보 목표:

- 권역당 raw candidate 300~500개
- 권역당 추천 가능 canonical place 100개 내외
- 전체 MVP 추천 가능 place 300개 내외

카테고리:

- 카페
- 음식점
- 술집/바
- 전시/문화공간
- 공연/행사
- 산책/관광명소
- 공방/체험
- 소품샵/쇼핑
- 사진/스튜디오

## DB 개념 모델

### places

우리 canonical place.

주요 필드:

- `id`
- `name`
- `normalized_name`
- `primary_category`
- `address`
- `road_address`
- `lat`
- `lon`
- `region_key`
- `operating_status`
- `curation_status`
- `created_at`
- `updated_at`

### place_sources

외부 source와의 연결 정보.

주요 필드:

- `id`
- `place_id`
- `source`
- `source_place_id`
- `source_url`
- `source_category`
- `source_name`
- `source_address`
- `matched_confidence`
- `last_verified_at`

source 예:

- `small_business`
- `localdata`
- `tourapi`
- `seoul_open_data`
- `kakao_local`
- `naver_local`
- `google_places`
- `owner_submitted`
- `user_submitted`

### place_candidates

수집되었지만 아직 canonical place로 승격되지 않은 후보.

주요 필드:

- `id`
- `source`
- `raw_payload`
- `name`
- `address`
- `lat`
- `lon`
- `source_category`
- `candidate_status`
- `dedupe_key`
- `created_at`

candidate status:

- `new`
- `matched`
- `rejected`
- `promoted`
- `needs_review`

### place_tags

데이트 추천에 쓰이는 자체 태그.

예:

- 조용한
- 감성적인
- 대화하기 좋은
- 사진 좋은
- 비 오는 날
- 기념일급
- 가성비
- 트렌디한
- 오래 머무는
- 걷기 좋은
- 실내
- 야외
- 체험형
- 술
- 디저트
- 전시

### user_place_feedback

사용자/커플 피드백.

주요 필드:

- `id`
- `couple_id`
- `place_id`
- `feedback_type`
- `created_at`

feedback type:

- `like`
- `dislike`

### place_quality_signals

추천 점수 계산용 내부 신호.

예:

- 좋아요율
- 싫어요율
- 저장률
- 클릭률
- 상세 조회율
- 외부 지도 이동률
- 신고 수
- 폐업 제보 수
- 검수 상태

### curation_reviews

사람이 검수한 기록.

주요 필드:

- `id`
- `place_id`
- `reviewer`
- `decision`
- `reason`
- `tag_notes`
- `reviewed_at`

decision:

- `approve`
- `reject`
- `needs_more_info`
- `merge`

## 중복 제거 전략

우선순위:

1. 동일 source의 고유 ID
2. 정규화된 장소명 + 도로명 주소
3. 정규화된 장소명 + 지번 주소
4. 정규화된 장소명 + 좌표 근접성
5. 전화번호

좌표 근접성 기준:

- 같은 이름 또는 매우 유사한 이름이면 30~50m 이내를 같은 장소 후보로 본다.
- 같은 건물 안의 서로 다른 매장 가능성이 있으므로 이름이 다르면 자동 merge하지 않는다.

정규화 예:

```text
카페 성수점 -> 카페
Cafe Seongsu -> cafe seongsu
공백/특수문자 제거
주식회사/유한회사/점/본점 등 suffix 처리
```

## 추천 가능 place 승격 기준

raw candidate가 바로 추천에 들어가면 품질이 낮아진다.

승격 기준:

- 서울 MVP 권역 안에 있음
- 데이트 가능 카테고리에 속함
- 주소/좌표가 안정적임
- 외부 source 1개 이상과 매칭됨
- 폐업/휴업 가능성이 낮음
- 중복이 정리됨
- 최소 태그가 부여됨
- 데이트 부적합 사유가 없음

추천 가능 상태:

```text
candidate
-> matched
-> curated
-> recommendable
```

## 초기 수집 파이프라인

```text
1. source ingest
   - 소상공인 상가정보 CSV/API
   - LOCALDATA 전체 다운로드
   - TourAPI
   - 서울 문화공간/문화행사

2. region filter
   - 성수, 연남/홍대, 한남/이태원 중심 좌표/radius 또는 polygon

3. category filter
   - 데이트 가능 업종은 넓게 포함
   - 병원/약국/학원/부동산/관공서 등 MVP 부적합 업종 제거

4. external enrichment
   - 카카오 로컬 API로 place id/url/category 보강
   - 네이버/구글은 현재성 및 보조 신호 확인

5. deduplication
   - source id
   - normalized name + address
   - normalized name + coordinate proximity

6. tagging
   - 룰 기반 1차 태그
   - LLM 기반 설명/카테고리 태깅
   - 사람이 핵심 후보 검수

7. promotion
   - candidate를 canonical place로 승격
   - recommendable 상태 부여

8. feedback loop
   - 호/불호 수집
   - 추천 점수 보정
   - 싫어요율 높은 장소 재검수
```

## 운영 업데이트 전략

주기:

- 소상공인 상가정보: 분기 단위 갱신 기준으로 재적재
- LOCALDATA: 초기 전체 다운로드 후 변동분 API/다운로드로 갱신
- TourAPI: 주기적 재수집
- 서울 문화행사: 매일 또는 주 1회 갱신
- 카카오/네이버/구글 보강: 핵심 권역 후보 중심으로 주기적 재검증

상태 관리:

- `active`
- `temporarily_closed`
- `closed`
- `unknown`
- `needs_reverification`

재검수 트리거:

- 외부 source에서 검색 안 됨
- 사용자 폐업 제보
- 싫어요율 급상승
- 주소/좌표 변경 감지
- 행사 종료일 경과

## 약관/저장 정책

안전한 원칙:

- 우리 DB의 중심은 공공데이터, 직접 큐레이션, 사용자/업주 제출 데이터로 둔다.
- 네이버/카카오/구글 응답은 원본 DB처럼 장기 저장하지 않는다.
- 외부 API 데이터는 매칭 ID, URL, 최소 참조 정보, 검증 시각 중심으로 저장한다.
- 리뷰 전문, 사진, 메뉴 이미지, 외부 플랫폼의 상세 콘텐츠는 저장하지 않는다.
- 광고 모델과 외부 지도 API 결과를 결합할 때는 약관/법무 검토를 한다.
- 캐시를 둘 경우 재검증 주기와 삭제 정책을 둔다.

## 사용자/업주 등록 플로우

long-tail 장소를 잡기 위해 초기부터 등록 플로우를 열어둔다.

사용자 제보:

- 장소명
- 위치/지도 링크
- 카테고리
- 데이트로 좋은 이유
- 추천 태그

업주 등록:

- 사업자명
- 상호명
- 주소
- 연락처
- 지도 링크
- 운영시간
- 대표 이미지
- 데이트 태그
- 예약/클래스 링크

검수 후 처리:

```text
submitted
-> matched with existing place
-> needs verification
-> approved
-> recommendable
```

## 초기 실행 순서

1. 첫 권역 3개 확정: 성수, 연남/홍대, 한남/이태원
2. 권역별 중심 좌표/radius 또는 polygon 정의
3. Vibe Dating 카테고리 8~10개 확정
4. 소상공인 업종코드 -> Vibe 카테고리 매핑
5. LOCALDATA 업종 -> Vibe 카테고리 매핑
6. 카카오 로컬 API 키워드 목록 확정
7. raw candidate 적재 테이블 설계
8. dedupe/promote 로직 설계
9. 권역별 raw candidate 300~500개 수집
10. 권역별 recommendable place 100개 수동 검수
11. 앱 MVP 추천에 연결
12. 호/불호 데이터로 점수 보정

## 다음에 정할 것

- 첫 권역별 정확한 경계
- 초기 추천 가능 카테고리
- 공방/체험 키워드 최종 목록
- 수동 검수 기준
- 외부 API 캐시/저장 정책
- canonical place schema
- 후보 수집 배치 구조
