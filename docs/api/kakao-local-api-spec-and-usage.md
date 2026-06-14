# Kakao Local API Spec And Usage

확인일: 2026-06-10

공식 문서:

- 로컬 API 이해하기: https://developers.kakao.com/docs/latest/ko/local/common
- 로컬 API REST API: https://developers.kakao.com/docs/latest/ko/local/dev-guide
- 쿼터/요금: https://developers.kakao.com/docs/latest/ko/getting-started/quota
- 운영정책: https://developers.kakao.com/terms/latest/ko/site-policies
- REST API 레퍼런스/에러: https://developers.kakao.com/docs/latest/ko/rest-api/reference

## 결론

카카오 로컬 API는 네이버 지역 검색 API보다 Vibe Dating의 장소 후보 확장에 더 적합하다.

좋은 점:

- 키워드 장소 검색과 카테고리 장소 검색을 모두 제공한다.
- 중심 좌표 + 반경 검색을 지원한다.
- 사각형 영역 검색을 지원한다.
- 페이지네이션을 지원한다.
- 음식점, 카페, 문화시설, 관광명소 같은 주요 카테고리 그룹 코드가 있다.
- 카카오맵 장소 상세 URL과 카카오 장소 ID를 제공한다.

한계:

- 평점, 리뷰 수, 방문자 리뷰, 사진, 메뉴, 영업시간은 제공하지 않는다.
- 카테고리 그룹이 크고 거칠다.
- 같은 query에서 노출 가능한 결과 수가 제한된다.
- 장기 저장/가공/광고 결합은 운영정책과 약관 검토가 필요하다.

권장 사용 위치:

```text
공공/오픈 데이터 seed
-> 카카오 로컬 API로 권역별 후보 보강
-> 카카오 place id / place url / 좌표 / 카테고리 매칭
-> 내부 카테고리와 데이트 태그로 재정제
```

## 준비물

카카오 로컬 API를 쓰려면 다음이 필요하다.

- 카카오디벨로퍼스 계정
- 카카오디벨로퍼스 앱 생성
- 앱의 `REST API 키`
- 앱 관리 페이지에서 `[카카오맵] > [사용 설정]` 상태를 `ON`으로 설정

주의:

- 2024-12-01부터 신규로 카카오맵 API를 호출하는 앱은 카카오맵 사용 설정이 필요하다.
- 서버에서 호출하는 것을 권장한다.
- 앱 키 유출 방지를 위해 호출 허용 IP 설정을 고려한다.

환경변수 예시:

```bash
KAKAO_REST_API_KEY=...
```

공통 인증 헤더:

```http
Authorization: KakaoAK ${KAKAO_REST_API_KEY}
```

## 주요 API 목록

| 기능 | Method | Endpoint | Vibe Dating 사용 |
| --- | --- | --- | --- |
| 키워드 장소 검색 | `GET` | `https://dapi.kakao.com/v2/local/search/keyword.json` | 권역 + 키워드 후보 수집 |
| 카테고리 장소 검색 | `GET` | `https://dapi.kakao.com/v2/local/search/category.json` | 음식점/카페/문화시설/관광명소 후보 수집 |
| 주소로 좌표 변환 | `GET` | `https://dapi.kakao.com/v2/local/search/address.json` | 공공데이터 주소 보정 |
| 좌표로 주소 변환 | `GET` | `https://dapi.kakao.com/v2/local/geo/coord2address.json` | 좌표 기반 주소 보정 |
| 좌표로 행정구역 변환 | `GET` | `https://dapi.kakao.com/v2/local/geo/coord2regioncode.json` | 위치의 행정동/법정동 확인 |
| 좌표계 변환 | `GET` | `https://dapi.kakao.com/v2/local/geo/transcoord.json` | 외부 데이터 좌표계 변환 |

응답 형식은 URL의 확장자로 `json` 또는 `xml`을 선택할 수 있다. 기본은 JSON으로 보면 된다.

## 키워드로 장소 검색

질의어에 매칭되는 장소를 검색한다. 특정 권역, 특정 데이트 카테고리 키워드를 조합할 때 사용한다.

```http
GET https://dapi.kakao.com/v2/local/search/keyword.json
```

### 요청 파라미터

| 파라미터 | 필수 | 설명 |
| --- | --- | --- |
| `query` | Y | 검색어. 예: `성수 카페`, `연남 와인바`, `한남 전시` |
| `category_group_code` | N | 카테고리 그룹 코드로 결과 필터링 |
| `x` | N | 중심 좌표 경도 |
| `y` | N | 중심 좌표 위도 |
| `radius` | N | 중심 좌표 기준 반경. 단위 m, 0~20000 |
| `rect` | N | 사각형 범위 검색. `좌측X,좌측Y,우측X,우측Y` |
| `page` | N | 페이지 번호. 1~45, 기본 1 |
| `size` | N | 페이지당 결과 수. 1~15, 기본 15 |
| `sort` | N | `accuracy` 또는 `distance`. 기본 `accuracy` |

`sort=distance`는 `x`, `y` 기준 좌표가 있어야 의미가 있다.

### 요청 예시

성수역 근처 2km 안에서 카페 검색:

```bash
curl -G "https://dapi.kakao.com/v2/local/search/keyword.json" \
  -H "Authorization: KakaoAK ${KAKAO_REST_API_KEY}" \
  --data-urlencode "query=성수 카페" \
  --data-urlencode "x=127.056655" \
  --data-urlencode "y=37.544581" \
  --data-urlencode "radius=2000" \
  --data-urlencode "size=15" \
  --data-urlencode "page=1" \
  --data-urlencode "sort=accuracy"
```

연남동 사각형 영역 안에서 와인바 검색:

```bash
curl -G "https://dapi.kakao.com/v2/local/search/keyword.json" \
  -H "Authorization: KakaoAK ${KAKAO_REST_API_KEY}" \
  --data-urlencode "query=와인바" \
  --data-urlencode "rect=126.9160,37.5540,126.9345,37.5685" \
  --data-urlencode "size=15" \
  --data-urlencode "page=1"
```

### 응답 주요 필드

```json
{
  "meta": {
    "total_count": 14,
    "pageable_count": 14,
    "is_end": true,
    "same_name": {
      "region": [],
      "keyword": "카페",
      "selected_region": ""
    }
  },
  "documents": [
    {
      "id": "26338954",
      "place_name": "장소명",
      "category_name": "음식점 > 카페",
      "category_group_code": "CE7",
      "category_group_name": "카페",
      "phone": "02-000-0000",
      "address_name": "서울 성동구 성수동...",
      "road_address_name": "서울 성동구 ...",
      "x": "127.05902969025047",
      "y": "37.51207412593136",
      "place_url": "http://place.map.kakao.com/26338954",
      "distance": "418"
    }
  ]
}
```

주의:

- `x`는 경도, `y`는 위도다.
- `distance`는 `x`, `y` 기준 좌표를 보낸 경우에만 내려온다.
- `meta.is_end=false`이면 `page`를 증가시켜 다음 페이지를 요청한다.
- `meta.pageable_count`는 노출 가능한 문서 수이고 공식 문서상 최대값이 45다. 따라서 한 query로 무한히 많이 가져오는 구조는 아니다.

## 카테고리로 장소 검색

미리 정의된 카테고리 그룹 코드에 해당하는 장소를 검색한다. 권역 안의 카페/음식점/문화시설/관광명소 후보를 넓게 가져올 때 쓴다.

```http
GET https://dapi.kakao.com/v2/local/search/category.json
```

### 요청 파라미터

| 파라미터 | 필수 | 설명 |
| --- | --- | --- |
| `category_group_code` | Y | 카테고리 그룹 코드 |
| `x` | 조건부 | 중심 좌표 경도. `x`,`y`,`radius` 조합 또는 `rect` 필요 |
| `y` | 조건부 | 중심 좌표 위도. `x`,`y`,`radius` 조합 또는 `rect` 필요 |
| `radius` | 조건부 | 중심 좌표 기준 반경. 단위 m, 0~20000 |
| `rect` | 조건부 | 사각형 범위 검색. `좌측X,좌측Y,우측X,우측Y` |
| `page` | N | 페이지 번호. 1~45, 기본 1 |
| `size` | N | 페이지당 결과 수. 1~15, 기본 15 |
| `sort` | N | `accuracy` 또는 `distance`. 기본 `accuracy` |

카테고리 검색은 `x`,`y`,`radius` 또는 `rect`가 필요하다고 보는 게 안전하다.

### 데이트 앱에 쓸 카테고리 그룹 코드

| 코드 | 카카오 카테고리 | Vibe Dating 용도 |
| --- | --- | --- |
| `FD6` | 음식점 | 맛집, 식사, 술집 일부 |
| `CE7` | 카페 | 카페, 디저트 |
| `CT1` | 문화시설 | 전시, 공연장, 미술관, 영화관 등 |
| `AT4` | 관광명소 | 산책, 랜드마크, 구경 코스 |
| `AD5` | 숙박 | MVP에서는 보통 제외 |
| `PK6` | 주차장 | 나중에 편의 정보 보강 |
| `SW8` | 지하철역 | 동선/권역 anchor |

카카오 카테고리에는 술집, 와인바, 공방, 팝업스토어 같은 세부 그룹이 따로 없으므로 이런 후보는 키워드 검색으로 보강해야 한다.

### 요청 예시

성수역 근처 2km 안의 카페:

```bash
curl -G "https://dapi.kakao.com/v2/local/search/category.json" \
  -H "Authorization: KakaoAK ${KAKAO_REST_API_KEY}" \
  --data-urlencode "category_group_code=CE7" \
  --data-urlencode "x=127.056655" \
  --data-urlencode "y=37.544581" \
  --data-urlencode "radius=2000" \
  --data-urlencode "size=15" \
  --data-urlencode "page=1" \
  --data-urlencode "sort=distance"
```

성수 권역 사각형 안의 문화시설:

```bash
curl -G "https://dapi.kakao.com/v2/local/search/category.json" \
  -H "Authorization: KakaoAK ${KAKAO_REST_API_KEY}" \
  --data-urlencode "category_group_code=CT1" \
  --data-urlencode "rect=127.0355,37.5330,127.0755,37.5585" \
  --data-urlencode "size=15" \
  --data-urlencode "page=1"
```

### 응답 필드

키워드 검색과 거의 동일하다.

- `id`
- `place_name`
- `category_name`
- `category_group_code`
- `category_group_name`
- `phone`
- `address_name`
- `road_address_name`
- `x`
- `y`
- `place_url`
- `distance`

## 주소/좌표 보조 API

### 주소로 좌표 변환

공공데이터나 수동 큐레이션 데이터에 주소만 있을 때 좌표를 보강한다.

```http
GET https://dapi.kakao.com/v2/local/search/address.json
```

주요 파라미터:

| 파라미터 | 필수 | 설명 |
| --- | --- | --- |
| `query` | Y | 검색할 주소 |
| `analyze_type` | N | `similar` 또는 `exact`, 기본 `similar` |
| `page` | N | 1~45, 기본 1 |
| `size` | N | 1~30, 기본 10 |

예시:

```bash
curl -G "https://dapi.kakao.com/v2/local/search/address.json" \
  -H "Authorization: KakaoAK ${KAKAO_REST_API_KEY}" \
  --data-urlencode "query=서울특별시 성동구 성수동"
```

### 좌표로 주소 변환

좌표만 있는 데이터에 지번/도로명 주소를 붙일 때 쓴다.

```http
GET https://dapi.kakao.com/v2/local/geo/coord2address.json
```

주요 파라미터:

| 파라미터 | 필수 | 설명 |
| --- | --- | --- |
| `x` | Y | 경도 |
| `y` | Y | 위도 |
| `input_coord` | N | 입력 좌표계. 기본 `WGS84` |

예시:

```bash
curl -G "https://dapi.kakao.com/v2/local/geo/coord2address.json" \
  -H "Authorization: KakaoAK ${KAKAO_REST_API_KEY}" \
  --data-urlencode "x=127.056655" \
  --data-urlencode "y=37.544581" \
  --data-urlencode "input_coord=WGS84"
```

### 좌표계 변환

외부 공공데이터 좌표계가 WGS84가 아닐 때 변환한다.

```http
GET https://dapi.kakao.com/v2/local/geo/transcoord.json
```

지원 좌표계:

- `WGS84`
- `WCONGNAMUL`
- `CONGNAMUL`
- `WTM`
- `TM`
- `KTM`
- `UTM`
- `BESSEL`
- `WKTM`
- `WUTM`

예시:

```bash
curl -G "https://dapi.kakao.com/v2/local/geo/transcoord.json" \
  -H "Authorization: KakaoAK ${KAKAO_REST_API_KEY}" \
  --data-urlencode "x=160710.37729270622" \
  --data-urlencode "y=-4388.879299157299" \
  --data-urlencode "input_coord=WTM" \
  --data-urlencode "output_coord=WGS84"
```

## 페이지네이션 방식

기본 로직:

```ts
async function fetchAllKakaoPages(fetchPage: (page: number) => Promise<KakaoSearchResponse>) {
  const all: KakaoPlaceDocument[] = [];

  for (let page = 1; page <= 45; page += 1) {
    const response = await fetchPage(page);
    all.push(...response.documents);

    if (response.meta.is_end) {
      break;
    }
  }

  return all;
}
```

주의:

- `page` 최대값은 45다.
- `size` 최대값은 장소 검색 기준 15다.
- 하지만 `pageable_count` 최대값이 45로 명시되어 있으므로 한 검색어/영역 조합에서 실질적으로 얻는 결과는 최대 45개로 보고 설계한다.
- 대량 수집은 하나의 큰 query가 아니라 권역, 카테고리, 키워드, 지도 타일을 쪼개는 방식으로 해야 한다.

## 내부 모델 정규화

카카오 응답을 내부 후보 장소 모델로 변환할 때의 기본 형태:

```ts
type KakaoLocalPlaceCandidate = {
  source: "kakao_local";
  sourcePlaceId: string;
  sourcePlaceUrl: string;
  name: string;
  categoryName: string;
  categoryGroupCode: string;
  categoryGroupName: string;
  phone: string;
  address: string;
  roadAddress: string;
  lon: number;
  lat: number;
  distanceMeters?: number;
};

function normalizeKakaoPlace(doc: KakaoPlaceDocument): KakaoLocalPlaceCandidate {
  return {
    source: "kakao_local",
    sourcePlaceId: doc.id,
    sourcePlaceUrl: doc.place_url,
    name: doc.place_name,
    categoryName: doc.category_name,
    categoryGroupCode: doc.category_group_code,
    categoryGroupName: doc.category_group_name,
    phone: doc.phone,
    address: doc.address_name,
    roadAddress: doc.road_address_name,
    lon: Number(doc.x),
    lat: Number(doc.y),
    distanceMeters: doc.distance ? Number(doc.distance) : undefined
  };
}
```

추천용 내부 카테고리는 카카오 카테고리를 그대로 쓰지 말고 별도로 둔다.

```text
kakao category_name/category_group_code
-> vibe category
-> vibe tags
```

예:

| Kakao | Vibe Category | 추가 태그 후보 |
| --- | --- | --- |
| `CE7`, `카페` | `cafe` | 대화, 디저트, 실내 |
| `FD6`, `음식점 > 양식` | `restaurant` | 식사, 기념일 가능 |
| `FD6`, `음식점 > 술집` | `bar` | 저녁, 술 |
| `CT1`, `문화시설 > 미술관` | `culture` | 전시, 실내 |
| `AT4`, `관광명소` | `walk_or_landmark` | 산책, 구경 |

## Vibe Dating 수집 전략

### 1. 권역 중심 좌표 기반 카테고리 검색

첫 MVP 권역마다 중심 좌표와 반경을 둔다.

예시:

| 권역 | 중심 좌표 예시 | 반경 |
| --- | --- | --- |
| 성수 | `127.056655,37.544581` | 1500~2500m |
| 연남/홍대 | `126.9237,37.5563` | 1500~2500m |
| 한남/이태원 | `127.0025,37.5345` | 1500~2500m |

수집 대상:

- `CE7`: 카페
- `FD6`: 음식점
- `CT1`: 문화시설
- `AT4`: 관광명소

### 2. 키워드 보강

카카오 카테고리 그룹만으로는 데이트 맥락이 부족하므로 키워드 검색을 병행한다.

권장 키워드:

- `카페`
- `디저트`
- `브런치`
- `와인바`
- `칵테일바`
- `이자카야`
- `술집`
- `전시`
- `미술관`
- `공방`
- `원데이클래스`
- `소품샵`
- `북카페`
- `루프탑`
- `팝업스토어`
- `산책`
- `공원`

검색 조합:

```text
{권역명} + {키워드}
중심 좌표 + radius + {키워드}
rect + {키워드}
```

예:

```text
성수 와인바
성수 공방
연남 디저트
한남 전시
을지로 칵테일바
```

### 3. 지도 타일 분할

한 query의 노출 결과가 제한되므로 큰 권역은 여러 타일로 쪼갠다.

```text
성수 전체 polygon
-> 500m~800m radius grid 생성
-> grid center별 category search
-> keyword search
-> kakao place id 기준 dedupe
```

너무 촘촘히 쪼개면 호출량과 중복이 커진다. MVP에서는 권역당 4~12개 grid 정도로 시작한다.

### 4. 중복 제거

우선순위:

1. `sourcePlaceId`, 즉 카카오 장소 ID
2. 정규화한 장소명 + 도로명 주소
3. 정규화한 장소명 + 좌표 근접성

좌표 근접성은 같은 이름 기준 30~50m 이내를 같은 장소 후보로 볼 수 있다.

### 5. 현재성 보정

카카오 응답에 장소가 나온다고 해서 데이트 장소로 적합하다는 뜻은 아니다.

추가 확인할 것:

- 공공데이터 seed와 매칭되는지
- 네이버/구글에도 존재하는지
- `place_url`이 살아있는지
- 전화번호/주소가 비어 있지 않은지
- 카테고리가 너무 엉뚱하지 않은지
- 성인/숙박/병원/학원 등 MVP에서 제외할 업종인지

## 쿼터와 비용

2026-06-10 확인 기준 공식 쿼터 페이지 내용:

- 전체 API 월간 무료 제공량: 3,000,000건
- 로컬 API 일간 무료 제공량:
  - 좌표로 주소 변환: 100,000건
  - 좌표로 행정구역정보 변환: 100,000건
  - 주소로 좌표 변환: 100,000건
  - 카테고리로 장소 검색: 100,000건
  - 키워드로 장소 검색: 100,000건
  - 좌표계 변환: 100,000건
- 추가 쿼터 요금:
  - 좌표/주소 변환 계열: 0.5원/건
  - 카테고리로 장소 검색: 2원/건
  - 키워드로 장소 검색: 2원/건

쿼터와 요금은 변경될 수 있으므로 실제 운영 전 다시 확인해야 한다.

대략적인 MVP 수집 호출량 예시:

```text
권역 3개
권역당 grid 8개
카테고리 4개
페이지 최대 3개

3 * 8 * 4 * 3 = 288 calls
```

키워드 보강:

```text
권역 3개
권역당 keyword 15개
페이지 최대 3개

3 * 15 * 3 = 135 calls
```

초기 seed 구축은 수백~수천 호출 수준으로 충분히 가능하다.

## 에러 처리

카카오 REST API는 실패 시 보통 JSON 형태로 `code`, `msg`를 반환한다.

대표적으로 처리할 것:

- `400`: 필수 파라미터 누락/잘못된 요청
- `401`: 인증 오류
- `403`: 권한 오류 또는 API 사용 설정 문제
- `429`: Too Many Request, 쿼터 또는 초당 요청 제한 초과
- `500/502/503`: 카카오 서버 오류 또는 점검

쿼터 초과 예시는 `code=-10`, `msg=API limit has been exceeded.` 형태로 안내된다.

운영 코드에서는 다음이 필요하다.

- timeout
- retry with backoff
- 429 발생 시 중단/지연
- 요청 파라미터와 응답 meta 로깅
- source별 fetch job id 기록

## 약관/운영정책 주의

카카오 운영정책에는 카카오에서 받은 데이터를 캐시할 때 사용자 환경 개선 목적과 최신성 유지가 중요하다는 취지의 제한이 있다. 또한 카카오에서 얻은 정보를 사전 승낙 없이 복사, 복제, 변경, 검색 엔진/디렉터리에 입력하거나 타인에게 제공하는 행위도 금지 행동으로 언급된다.

따라서 Vibe Dating에서는 다음처럼 접근하는 것이 안전하다.

- 카카오 응답을 자체 장소 DB의 유일한 원본으로 삼지 않는다.
- 공공데이터/수동 큐레이션/사용자 피드백을 우리 DB의 중심으로 둔다.
- 카카오 데이터는 후보 보강, 장소 매칭, 상세 URL 연결, 현재성 확인에 우선 사용한다.
- 카카오 장소 정보를 장기 저장/가공/광고 모델과 결합하려면 약관/법무/제휴 검토를 별도로 한다.
- 캐시를 둘 경우 재검증 주기와 삭제 정책을 둔다.

## MVP 권장 구조

```text
1. 공공데이터 seed 적재
   - 소상공인 상가 정보
   - TourAPI
   - 서울 문화공간/문화행사

2. 카카오 로컬 API 보강
   - category search: CE7, FD6, CT1, AT4
   - keyword search: 와인바, 공방, 전시, 디저트 등
   - address/geocode 보정

3. 정규화
   - kakao place id
   - place_url
   - name/address/coords/category
   - vibe category/tag

4. 중복 제거
   - sourcePlaceId
   - normalized name + road address
   - normalized name + nearby coordinates

5. 수동 검수
   - 권역별 top 후보
   - 데이트 부적합 장소 제거
   - 감성/상황 태그 부여

6. 추천 스코어링
   - 커플 태그 매칭
   - 거리/권역
   - 카테고리 선호
   - 호/불호 누적
```

## 간단한 Node.js 호출 예시

```ts
const KAKAO_REST_API_KEY = process.env.KAKAO_REST_API_KEY;

type KakaoSearchResponse = {
  meta: {
    total_count: number;
    pageable_count: number;
    is_end: boolean;
  };
  documents: KakaoPlaceDocument[];
};

type KakaoPlaceDocument = {
  id: string;
  place_name: string;
  category_name: string;
  category_group_code: string;
  category_group_name: string;
  phone: string;
  address_name: string;
  road_address_name: string;
  x: string;
  y: string;
  place_url: string;
  distance?: string;
};

async function searchKakaoKeyword(params: Record<string, string | number>) {
  const url = new URL("https://dapi.kakao.com/v2/local/search/keyword.json");

  Object.entries(params).forEach(([key, value]) => {
    url.searchParams.set(key, String(value));
  });

  const response = await fetch(url, {
    headers: {
      Authorization: `KakaoAK ${KAKAO_REST_API_KEY}`
    }
  });

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`Kakao Local API failed: ${response.status} ${body}`);
  }

  return response.json() as Promise<KakaoSearchResponse>;
}

const result = await searchKakaoKeyword({
  query: "성수 카페",
  x: 127.056655,
  y: 37.544581,
  radius: 2000,
  size: 15,
  page: 1,
  sort: "accuracy"
});
```

## 다음 작업 후보

- 첫 MVP 권역 3개 확정
- 권역별 중심 좌표/radius/rect 정의
- 권역별 keyword 목록 확정
- `Kakao category_group_code -> Vibe category` 매핑 테이블 작성
- 수집 job 설계
- 캐시/저장/재검증 정책 설계
