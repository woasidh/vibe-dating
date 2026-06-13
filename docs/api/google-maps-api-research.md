# Google Maps API Research

작성일: 2026-06-10

## 결론

Vibe Dating MVP에는 **Google Maps JavaScript API + Places API(New)** 조합이 가장 직접적이다.

- 지도 표시: Maps JavaScript API
- 지도 핀: Advanced Marker
- 장소 검색/후보 수집: Places API(New)의 Text Search, Nearby Search
- 장소 상세: Place Details
- 주소와 좌표 변환: Geocoding API
- 코스, 거리, 동선: 후순위로 Routes API 또는 Route Matrix

다만 서울/한국 서비스라면 Google Maps 단독 의존은 위험하다. Google 공식 커버리지 표에서 South Korea는 2D 지도와 Geocoding은 가능하지만 Traffic Layer, Driving Directions, Biking Directions, Walking Directions, Speed Limits 쪽은 미지원 또는 낮은 품질로 표시된다. MVP의 장소 표시 UI는 Google로 시작할 수 있지만, 동선/길찾기/국내 장소 데이터 품질은 네이버 지도 또는 카카오맵과 비교해야 한다.

## 초기 세팅

1. Google Cloud 프로젝트 생성
2. Billing 연결
3. 필요한 API enable
   - Maps JavaScript API
   - Places API
   - 필요 시 Geocoding API
   - 필요 시 Routes API
4. API key 생성
5. API key 제한 설정

권장 키 분리:

- 프론트 지도용 키
  - Application restriction: HTTP referrer
  - API restriction: Maps JavaScript API, 필요한 경우 Places Library
- 서버 Places/Geocoding 호출용 키
  - Application restriction: 서버 IP
  - API restriction: Places API, Geocoding API

프론트에서 쓰는 Google Maps JavaScript API key는 브라우저에 노출되는 구조다. 숨기는 것보다 referrer 제한과 API 제한을 제대로 거는 것이 핵심이다.

## 웹에서 붙이는 방식

Maps JavaScript API 로딩 방식은 세 가지가 있다.

- Dynamic Library Import: 필요한 라이브러리만 런타임에 로드
- Direct script tag: 단순하지만 한 번에 로드
- npm `js-api-loader`: 번들러 환경에서 관리하기 편함

React/Next.js라면 `@vis.gl/react-google-maps` 사용이 가장 무난하다. Google 공식 React codelab도 이 라이브러리를 사용한다.

```tsx
import { APIProvider, AdvancedMarker, Map } from "@vis.gl/react-google-maps";

export function DateSpotMap() {
  return (
    <APIProvider apiKey={process.env.NEXT_PUBLIC_GOOGLE_MAPS_API_KEY!}>
      <Map
        mapId={process.env.NEXT_PUBLIC_GOOGLE_MAP_ID}
        defaultCenter={{ lat: 37.5665, lng: 126.978 }}
        defaultZoom={13}
      >
        <AdvancedMarker position={{ lat: 37.5665, lng: 126.978 }} />
      </Map>
    </APIProvider>
  );
}
```

Advanced Marker를 쓰려면 `mapId`가 필요하다. 테스트 중에는 `DEMO_MAP_ID`를 쓸 수 있지만, 프로덕션에서는 Google Cloud Console에서 직접 만든 Map ID를 쓰는 편이 좋다.

## Places API 주요 스펙

### Text Search

사용처:

- "성수동 감성 카페"
- "홍대 데이트하기 좋은 술집"
- "restaurants in Gangnam"

Endpoint:

```text
POST https://places.googleapis.com/v1/places:searchText
```

특징:

- `textQuery` 필요
- `X-Goog-FieldMask` 필요
- 결과는 최대 60개 across pages
- 같은 요청이어도 결과 일관성이 보장되지는 않음

예시:

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "X-Goog-Api-Key: API_KEY" \
  -H "X-Goog-FieldMask: places.id,places.displayName,places.formattedAddress,places.location" \
  -d '{
    "textQuery": "성수동 감성 카페",
    "languageCode": "ko",
    "regionCode": "KR"
  }' \
  "https://places.googleapis.com/v1/places:searchText"
```

### Nearby Search

사용처:

- 현재 위치 주변 카페
- 선택 권역 반경 내 음식점
- 지도 중심점 주변 장소 후보

Endpoint:

```text
POST https://places.googleapis.com/v1/places:searchNearby
```

특징:

- `locationRestriction.circle` 필요
- radius는 `0 ~ 50000m`
- `maxResultCount`는 `1 ~ 20`
- `includedTypes`, `excludedTypes`, `includedPrimaryTypes`, `excludedPrimaryTypes`로 필터 가능
- `rankPreference`는 `POPULARITY` 또는 `DISTANCE`

예시:

```bash
curl -X POST \
  -H "Content-Type: application/json" \
  -H "X-Goog-Api-Key: API_KEY" \
  -H "X-Goog-FieldMask: places.id,places.displayName,places.formattedAddress,places.location,places.primaryType" \
  -d '{
    "includedTypes": ["restaurant"],
    "maxResultCount": 10,
    "locationRestriction": {
      "circle": {
        "center": {
          "latitude": 37.5445,
          "longitude": 127.0557
        },
        "radius": 1500
      }
    },
    "languageCode": "ko",
    "regionCode": "KR"
  }' \
  "https://places.googleapis.com/v1/places:searchNearby"
```

### Place Details

사용처:

- Place ID로 상세 정보 조회
- 추천 카드 렌더링용 이름, 주소, 위치, 평점, 사진 등 조회

Endpoint:

```text
GET https://places.googleapis.com/v1/places/{PLACE_ID}
```

특징:

- `FieldMask` 필수
- 요청 필드에 따라 SKU가 달라짐
- 필요 필드만 요청해야 비용과 응답 크기를 줄일 수 있음

예시:

```bash
curl -X GET \
  -H "Content-Type: application/json" \
  -H "X-Goog-Api-Key: API_KEY" \
  -H "X-Goog-FieldMask: id,displayName,formattedAddress,location,rating,userRatingCount,googleMapsUri" \
  "https://places.googleapis.com/v1/places/PLACE_ID"
```

## 비용 감각

Google Maps Platform은 SKU별 pay-as-you-go 모델이다. 공식 가격표 기준 주요 항목은 다음과 같다.

| SKU | 무료 사용량 | 이후 가격 |
| --- | ---: | ---: |
| Dynamic Maps | 월 10,000 map loads | $7 / 1,000 |
| Geocoding | 월 10,000 events | $5 / 1,000 |
| Autocomplete Requests | 월 10,000 requests | $2.83 / 1,000 |
| Place Details Essentials | 월 10,000 requests | $5 / 1,000 |
| Nearby Search Pro | 월 5,000 requests | $32 / 1,000 |
| Text Search Pro | 월 5,000 requests | $32 / 1,000 |
| Routes Compute Routes Essentials | 월 10,000 requests | $5 / 1,000 |
| Routes Compute Route Matrix Essentials | 월 10,000 elements | $5 / 1,000 |

장소명, 주소, 좌표, 타입 등 실제 리스트 구성에 필요한 필드는 검색 API에서 Pro SKU로 올라갈 수 있다. 따라서 장소 후보를 대량으로 긁어오는 방식은 비용이 빠르게 커질 수 있다.

## FieldMask 전략

Places API(New)는 Text Search, Nearby Search, Place Details에서 FieldMask가 중요하다.

- FieldMask를 생략하면 에러가 난다.
- `*`는 개발 중 테스트에는 가능하지만 프로덕션에서는 지양해야 한다.
- 필요한 필드만 요청해야 비용과 latency를 줄일 수 있다.
- Essentials/Pro/Enterprise 필드를 섞으면 가장 높은 SKU 기준으로 과금될 수 있다.

MVP 추천 필드:

```text
places.id,
places.displayName,
places.formattedAddress,
places.location,
places.primaryType,
places.types,
places.googleMapsUri
```

평점 기반 스코어링이 필요하면 아래 필드가 필요하지만, SKU 상승 가능성을 확인해야 한다.

```text
places.rating,
places.userRatingCount
```

## 약관과 저장 정책

Places API 콘텐츠는 pre-fetch, cache, store가 제한된다. 예외적으로 **Place ID는 장기 저장 가능**하다.

따라서 Vibe Dating에서 Google Places를 데이터 원천처럼 사용해 장소명, 주소, 평점, 리뷰, 사진 등을 장기 저장하는 방식은 위험하다.

권장 구조:

- 내부 DB에 저장
  - `google_place_id`
  - `naver_place_id`
  - 자체 카테고리
  - 자체 데이트 태그
  - 자체 큐레이션 점수
  - 유저 호/불호 데이터
- 필요 시 실시간 조회
  - Google 장소명
  - Google 주소
  - Google 평점
  - Google 사진
  - Google Maps URL

Google Places 데이터를 지도 위에 표시할 경우 Google Map 위에 표시하고 attribution을 유지해야 한다. Google Map 없이 Places 데이터를 보여주는 경우에도 Google Maps attribution 요구사항을 확인해야 한다.

## Vibe Dating MVP 적용 방향

추천 아키텍처:

1. 자체 장소 DB를 만든다.
2. Google Place ID와 Naver Place ID를 매칭 키로 저장한다.
3. 지도 UI는 Google Maps JavaScript API로 구현한다.
4. 추천 핀은 Advanced Marker로 표시한다.
5. 장소 후보 발굴 단계에서 Google Places와 네이버 지도 API를 비교한다.
6. Google Places 데이터는 장기 저장하지 않고 필요 시 보강 조회한다.
7. 코스 자동 생성, 길찾기, 동선 최적화는 한국 커버리지 확인 후 후순위로 둔다.

MVP에서 좋은 첫 화면:

- 지역/현재 위치 선택
- 성향 태그 선택
- 추천 장소 리스트
- 지도 핀
- 장소별 호/불호

초기에는 지도 편집/자동 코스 생성보다 "추천 장소를 지도와 리스트로 잘 보여주는 것"에 집중하는 편이 낫다.

## 리스크

- Google Places Search는 비용이 빠르게 증가할 수 있다.
- 한국 길찾기/동선 기능은 Google 커버리지 한계가 있다.
- Places 데이터 장기 저장은 약관 리스크가 있다.
- Google/Naver 중복 제거와 카테고리 정규화가 필요하다.
- 데이트 감성, 분위기, 웨이팅 싫음 같은 정보는 API만으로 판단하기 어렵다.

## 다음 액션

1. 서울 첫 타깃 권역을 정한다.
2. 각 권역별 Google Places와 네이버 지도 결과 품질을 비교한다.
3. 자체 장소 스키마를 설계한다.
4. `google_place_id`, `naver_place_id`, 자체 태그를 분리해서 저장한다.
5. 지도 MVP는 Google Maps JavaScript API + Advanced Marker로 붙인다.
6. API key 제한과 월 예산 알림을 먼저 설정한다.

## 참고 문서

- [Getting started with Google Maps Platform](https://developers.google.com/maps/get-started)
- [Maps JavaScript API usage and billing](https://developers.google.com/maps/documentation/javascript/usage-and-billing)
- [Load the Maps JavaScript API](https://developers.google.com/maps/documentation/javascript/load-maps-js-api)
- [Advanced Markers](https://developers.google.com/maps/documentation/javascript/advanced-markers/start)
- [Places API usage and billing](https://developers.google.com/maps/documentation/places/web-service/usage-and-billing)
- [Nearby Search(New)](https://developers.google.com/maps/documentation/places/web-service/nearby-search)
- [Text Search(New)](https://developers.google.com/maps/documentation/places/web-service/text-search)
- [Place Details(New)](https://developers.google.com/maps/documentation/places/web-service/place-details)
- [Choose fields to return](https://developers.google.com/maps/documentation/places/web-service/choose-fields)
- [Place IDs](https://developers.google.com/maps/documentation/places/web-service/place-id)
- [Places API policies and attributions](https://developers.google.com/maps/documentation/places/web-service/policies)
- [Google Maps Platform pricing list](https://developers.google.com/maps/billing-and-pricing/pricing)
- [API security best practices](https://developers.google.com/maps/api-security-best-practices)
- [Google Maps Platform coverage details](https://developers.google.com/maps/coverage)
