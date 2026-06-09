# Naver Local API And Place Data Strategy

확인일: 2026-06-10

## 결론

네이버에서 바로 쓸 수 있는 공개 API는 흔히 말하는 "네이버 플레이스 상세 API"가 아니라 `검색 API > 지역`이다.

이 API는 실시간 장소 검색 보조에는 쓸 수 있지만, Vibe Dating의 데이트 장소 후보를 대량으로 수집하고 자체 추천 DB를 만드는 메인 소스로 쓰기에는 애매하다.

핵심 이유:

- 한 번에 최대 5개만 반환한다.
- `start`가 최대 1이라 사실상 페이지네이션이 안 된다.
- 반경 검색이 없다.
- 평점, 방문자 리뷰, 영업시간, 사진, 메뉴 같은 플레이스 상세 데이터가 없다.
- 네이버 지역정보를 별도 DB로 관리하거나 광고 모델에 붙이는 것은 약관 검토가 필요하다.

따라서 권장 방향은 다음과 같다.

```text
공공/오픈 데이터로 대량 seed 구축
+ 지도 API로 좌표/상세 URL/현재성 보정
+ 사용자 호/불호 데이터로 추천 품질 강화
+ 일부 핵심 권역은 수동 큐레이션
```

## 네이버 지역 검색 API

공식 문서:

- https://developers.naver.com/docs/serviceapi/search/local/local.md

### 준비물

- 네이버 개발자센터 계정
- Application 등록
- 사용 API에서 `검색` 권한 활성화
- `NAVER_CLIENT_ID`
- `NAVER_CLIENT_SECRET`

서버 환경변수 예시:

```bash
NAVER_CLIENT_ID=...
NAVER_CLIENT_SECRET=...
```

프론트엔드에서 직접 호출하면 `Client Secret`이 노출되므로, 반드시 백엔드에서 프록시 형태로 호출한다.

### 요청 스펙

```http
GET https://openapi.naver.com/v1/search/local.json
```

헤더:

```http
X-Naver-Client-Id: {client_id}
X-Naver-Client-Secret: {client_secret}
```

쿼리 파라미터:

| 파라미터 | 필수 | 설명 |
| --- | --- | --- |
| `query` | Y | 검색어. 예: `성수 카페`, `연남 데이트`, `한남 와인바` |
| `display` | N | 결과 개수. 기본 1, 최대 5 |
| `start` | N | 검색 시작 위치. 기본 1, 최대 1 |
| `sort` | N | `random`: 정확도순, `comment`: 카페/블로그 리뷰 개수순 |

요청 예시:

```bash
curl "https://openapi.naver.com/v1/search/local.json?query=%EC%84%B1%EC%88%98%20%EC%B9%B4%ED%8E%98&display=5&start=1&sort=random" \
  -H "X-Naver-Client-Id: ${NAVER_CLIENT_ID}" \
  -H "X-Naver-Client-Secret: ${NAVER_CLIENT_SECRET}"
```

### 응답 필드

```json
{
  "lastBuildDate": "...",
  "total": 407,
  "start": 1,
  "display": 5,
  "items": [
    {
      "title": "장소명",
      "link": "상세 URL",
      "category": "음식점>카페",
      "description": "설명",
      "telephone": "",
      "address": "지번 주소",
      "roadAddress": "도로명 주소",
      "mapx": "1269873882",
      "mapy": "375666103"
    }
  ]
}
```

좌표는 WGS84 기준이며, 응답값은 보통 `1e7` 스케일로 내려온다.

```ts
const lon = Number(item.mapx) / 10_000_000;
const lat = Number(item.mapy) / 10_000_000;
```

내부 후보 모델로 정규화할 때는 아래 정도만 안정적으로 기대한다.

```ts
type NaverLocalPlaceCandidate = {
  source: "naver_local";
  sourcePlaceUrl: string;
  name: string;
  categoryText: string;
  description: string;
  address: string;
  roadAddress: string;
  lon: number;
  lat: number;
};
```

## 네이버 지도 표시

공식 문서:

- https://navermaps.github.io/maps.js.ncp/docs/tutorial-2-Getting-Started.html

준비물:

- 네이버 클라우드 플랫폼 계정
- Application Services > Maps 애플리케이션 등록
- `Dynamic Map` 선택
- 웹 서비스 URL 등록
- Maps Client ID, 즉 `ncpKeyId`

지도 로드 예시:

```html
<script src="https://oapi.map.naver.com/openapi/v3/maps.js?ncpKeyId=YOUR_CLIENT_ID"></script>

<div id="map" style="width:100%;height:400px;"></div>

<script>
  const map = new naver.maps.Map("map", {
    center: new naver.maps.LatLng(37.5447, 127.0557),
    zoom: 14
  });

  new naver.maps.Marker({
    position: new naver.maps.LatLng(37.5447, 127.0557),
    map
  });
</script>
```

## 네이버 Geocoding

공식 문서:

- https://api.ncloud-docs.com/docs/ainaverapi-maps-overview

주소를 좌표로 변환해야 할 때 사용한다.

```http
GET https://naveropenapi.apigw.ntruss.com/map-geocode/v2/geocode
```

헤더:

```http
X-NCP-APIGW-API-KEY-ID: {NCP Maps Client ID}
X-NCP-APIGW-API-KEY: {NCP Maps Client Secret}
```

요청 예시:

```bash
curl "https://naveropenapi.apigw.ntruss.com/map-geocode/v2/geocode?query=서울특별시%20성동구%20성수동" \
  -H "X-NCP-APIGW-API-KEY-ID: ${NCP_MAPS_KEY_ID}" \
  -H "X-NCP-APIGW-API-KEY: ${NCP_MAPS_KEY}"
```

## 네이버 API 사용 위치

Vibe Dating에서는 네이버 API를 아래 용도로 제한하는 것이 현실적이다.

- 사용자가 검색한 장소명을 실시간으로 확인한다.
- 추천 장소를 네이버 지도 상세 페이지로 연결한다.
- 지도 위에 후보 장소를 표시한다.
- 내부 DB에 있는 장소와 네이버 검색 결과를 매칭해 현재성을 확인한다.

대신 아래 용도로는 부적합하다.

- 서울 전체 데이트 장소 대량 수집
- 자체 장소 DB의 주 데이터 소스
- 평점/리뷰/사진/영업시간 기반 스코어링
- 광고 모델과 결합된 네이버 지역정보 DB 운영

## 대량 데이트 장소 확보 전략

### 1. 소상공인 상가 정보로 음식점/카페/술집 seed 구축

공식 데이터:

- https://www.data.go.kr/data/15012005/openapi.do
- https://www.data.go.kr/data/15083033/fileData.do

장점:

- 전국 단위 영업 중 상가업소 데이터다.
- 상호명, 업종코드, 업종명, 지번주소, 도로명주소, 경도, 위도 등을 제공한다.
- CSV 파일 다운로드와 OpenAPI를 모두 제공한다.
- 비용이 무료이고 이용허락범위 제한 없음으로 표시되어 있다.
- 서울 핫플 권역의 음식점, 카페, 주점 후보를 대량으로 뽑기에 가장 좋은 seed다.

한계:

- 데이트 적합도, 분위기, 평점, 리뷰, 사진은 없다.
- 업종 분류가 데이트 맥락에 맞게 정제되어 있지 않다.
- 폐업/이전/상호 변경은 보정이 필요하다.

권장 사용:

```text
상가 CSV/API 수집
-> 서울/핫플 권역 필터
-> 업종코드 기반으로 음식점/카페/주점/디저트 추출
-> 네이버/카카오/구글 검색 API로 존재 여부 보정
-> 내부 데이트 태그 수동/반자동 부여
```

### 2. 한국관광공사 TourAPI로 관광지/문화시설/행사 seed 구축

공식 데이터:

- https://www.data.go.kr/data/15101578/openapi.do

장점:

- 관광지, 문화시설, 축제/공연/행사, 여행코스, 레포츠, 숙박, 쇼핑, 음식점 계열 데이터를 다룰 수 있다.
- JSON/XML REST API를 제공한다.
- 비용 무료이고 개발계정은 자동승인이다.
- 이미지/소개/상세 정보 계열 endpoint를 함께 활용할 수 있다.
- 데이트에서 중요한 전시, 팝업성 행사, 관광명소, 산책 후보를 만들기 좋다.

한계:

- 일반 핫플 카페/식당 커버리지는 약할 수 있다.
- 최신 로컬 트렌드 반영은 지도/검색 API나 수동 큐레이션이 필요하다.

권장 사용:

```text
TourAPI areaBasedList/locationBasedList/searchFestival/searchKeyword 수집
-> 서울 contentType 필터
-> 관광지/문화시설/행사/음식점 후보 분리
-> 이미지와 소개 정보 보강
-> 날짜가 있는 행사는 만료/진행중 상태 관리
```

### 3. 서울 열린데이터광장으로 문화공간/문화행사 보강

공식 데이터 예시:

- 서울시 문화공간 정보: https://data.seoul.go.kr/dataList/OA-15487/A/1/datasetView.do
- 서울시 문화행사 정보: https://data.seoul.go.kr/dataList/datasetView.do?infId=OA-15486&serviceKind=1&srvType=S

장점:

- 서울 MVP와 잘 맞는다.
- 문화공간은 시설명, 주소, 위경도, 소개, 홈페이지, 관람시간, 관람료, 휴관일 등을 제공한다.
- 문화행사는 공연/행사명, 장소, 날짜, 이용요금, 위치, 시작일/종료일 등을 제공한다.
- 데이트 장소 중 전시, 공연, 문화공간, 실내 데이트 후보를 보강하기 좋다.

한계:

- 카페/음식점/술집은 커버하지 못한다.
- 행사 데이터는 날짜 만료 관리가 필요하다.

### 4. 카카오 로컬 API로 권역별 보강 검색

공식 문서:

- https://developers.kakao.com/docs/ko/local/dev-guide

장점:

- 키워드 장소 검색과 카테고리 장소 검색을 제공한다.
- 반경 검색, 사각형 영역 검색, 페이지네이션을 제공한다.
- 카테고리 그룹에 `CT1` 문화시설, `AT4` 관광명소, `FD6` 음식점, `CE7` 카페 등이 있다.
- 네이버 지역 검색보다 MVP 후보 확장에는 유리하다.

주요 스펙:

| API | Endpoint | 특징 |
| --- | --- | --- |
| 키워드 검색 | `/v2/local/search/keyword.json` | `query`, `x`, `y`, `radius`, `rect`, `page`, `size`, `sort` |
| 카테고리 검색 | `/v2/local/search/category.json` | `category_group_code`, `x`, `y`, `radius`, `rect`, `page`, `size`, `sort` |

페이지/크기:

- `page`: 1~45
- `size`: 1~15
- `radius`: 최대 20,000m

응답 필드:

- 장소 ID
- 장소명
- 카테고리
- 전화번호
- 지번/도로명 주소
- 좌표
- 카카오맵 상세 URL
- 기준 좌표와의 거리

주의:

- 카카오 데이터도 장기 저장/가공/광고 결합은 약관 검토가 필요하다.
- 실시간 보강/검증 용도로 시작하는 것이 안전하다.

### 5. Google Places API로 글로벌/리뷰 신호 보강

공식 문서:

- https://developers.google.com/maps/documentation/places/web-service/reference/rest/v1/places/searchText

장점:

- Text Search, Nearby Search, Place Details 등을 통해 장소 상세 신호를 얻을 수 있다.
- rating, review count, price level, open now, place type 같은 추천 스코어링용 신호가 있다.
- 외국인 사용자/영문 검색까지 고려하면 장기적으로 가치가 있다.

한계:

- 한국 로컬 장소 커버리지는 네이버/카카오보다 약할 수 있다.
- 유료 과금과 필드 마스크 설계가 중요하다.
- Google Maps Content 저장/재배포 제한도 약관 검토가 필요하다.
- Text Search는 페이지당 최대 20개, 전체 페이지 합산 최대 60개 제한이 있다.

권장 사용:

```text
내부 후보 장소명 + 주소
-> Google Places Text Search/Details로 매칭
-> rating/userRatingCount/priceLevel/types 정도만 scoring feature로 활용
-> 원본 데이터 저장 정책은 약관 기준으로 별도 관리
```

### 6. 수동 큐레이션과 사용자 피드백

데이트 장소 추천은 단순 POI 수집만으로 완성되지 않는다.

초기에는 아래 방식이 필요하다.

- 성수, 연남, 한남, 을지로 같은 첫 권역별로 사람이 직접 top 후보를 50~100개 선별한다.
- 각 장소에 내부 태그를 붙인다.
- 예: 조용한, 감성적인, 사진 좋은, 대화 중심, 걷기 좋은, 기념일급, 비 오는 날, 웨이팅 부담
- 사용자 호/불호 데이터를 누적해 태그와 카테고리 가중치를 보정한다.

## 추천 데이터 파이프라인 v1

```text
1. Seed 수집
   - 소상공인 상가 CSV/API
   - TourAPI
   - 서울 열린데이터광장

2. 후보 정제
   - 서울 핫플 권역 경계 필터
   - 업종/카테고리 필터
   - 폐업 가능성 높은 후보 제거
   - 좌표/주소 정규화

3. 지도 API 보강
   - 카카오/네이버/구글에서 장소명+주소 매칭
   - 외부 상세 URL 확보
   - 현재성/중복 확인

4. 내부 태그 부여
   - 규칙 기반 태그
   - LLM 기반 설명/카테고리 태깅
   - 핵심 권역 수동 검수

5. 추천 스코어링
   - 커플 성향 태그 매칭
   - 거리/권역
   - 카테고리 선호도
   - 외부 신뢰 신호
   - 사용자 호/불호 누적

6. 운영
   - 주기적 재수집
   - 후보 만료/폐업/중복 관리
   - 수동 큐레이션 큐 운영
```

## MVP 기준 추천 조합

1차 MVP에서는 아래 조합이 가장 현실적이다.

```text
소상공인 상가 정보
+ TourAPI
+ 서울 문화공간/문화행사 데이터
+ 카카오 로컬 API 보강
+ 네이버 지도 표시/상세 링크 보조
+ 핵심 권역 수동 큐레이션
```

이 조합의 장점:

- 대량 seed를 합법적이고 저렴하게 확보할 수 있다.
- 서울 MVP와 잘 맞는다.
- 음식점/카페/술집/문화공간/행사 후보를 모두 커버할 수 있다.
- 네이버 API의 페이지네이션 한계를 피할 수 있다.
- 추천 품질은 내부 태그와 사용자 피드백으로 개선할 수 있다.

## 다음 작업 후보

- 서울 첫 권역 3개 선정
- 권역 polygon 또는 중심 좌표/radius 정의
- 데이트 장소 카테고리 체계 정의
- 소상공인 업종코드 -> Vibe Dating 카테고리 매핑
- TourAPI contentType -> Vibe Dating 카테고리 매핑
- 후보 장소 deduplication key 설계
- 외부 API 원본 데이터 저장 정책/약관 체크
