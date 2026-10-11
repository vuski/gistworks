# Google 지도를 MapLibre로 교체

- 날짜: 2026-10-11
- 대상: `chungnamcompany`, `seoulcompany`, `foodscapeSeoul`

## 배경

세 프로젝트는 Google Maps JavaScript API를 사용했고, API 키(`AIza…`)가 HTML에 그대로 노출되어 있었다.
API 키가 필요 없는 무료 지도로 바꾸고, 키는 코드에서 삭제했다.
노출되었던 키는 Google Cloud Console에서 삭제했다. 다만 git 기록에는 남아 있다.

나머지 프로젝트는 d3만 사용하므로 변경하지 않았다.

## 변경 내용

### 라이브러리와 베이스맵

- Google Maps 스크립트를 `maplibre-gl@4.7.1`(unpkg CDN)로 교체했다.
- 베이스맵은 API 키가 필요 없는 Carto GL 스타일을 사용한다.

| 프로젝트 | 원래 지도 | 교체한 베이스맵 |
|---|---|---|
| chungnamcompany | Google 지도, 채도 -100 (흑백) | Carto Positron (회색조) |
| seoulcompany | Google 지도, 채도 -100 (흑백) | Carto Positron (회색조) |
| foodscapeSeoul | Google 기본 지도(ROADMAP) | Carto Voyager |

- Carto 스타일은 낮은 줌에서 영문 이름(`name_en`)을 표시한다. 그래서 지도가 로드된 뒤 `applyKoreanLabels()`를 호출해, 모든 symbol 레이어의 `text-field`를 `coalesce(name:ko, name)`으로 바꾼다. 한글 글리프는 MapLibre가 브라우저 로컬 폰트로 렌더링한다.
- 줌 단계는 Google보다 1 낮게 설정했다(Google 11 → MapLibre 10, Google 13 → MapLibre 12). 타일 크기가 256px와 512px로 다르기 때문에, 이렇게 해야 같은 축척이 된다.
- 확대/축소 버튼(NavigationControl)을 추가했다. foodscapeSeoul에서는 도넛 차트와 겹치지 않도록 왼쪽 아래에 두었다.

### 격자 폴리곤

- 격자마다 하나씩 만들던 `google.maps.Polygon` 수천 개를 GeoJSON 소스 하나(`cells`)와 fill/line 레이어로 바꿨다.
- 반투명 흰색 배경 사각형(`DrawBackground`)은 `background` 소스와 fill 레이어로 옮겼다. 불투명도 0.8은 그대로 두었다.
- 레이어 순서도 원래와 같다. 배경 사각형과 격자가 베이스맵 라벨보다 위에 그려진다.

### chungnamcompany / seoulcompany 전용

- 색칠 로직(`reInit`, E 키, 스페이스 키)은 그대로 두었다. 대신 각 `recs[i].polygon`에 `makePolygonHandle(id)`가 반환하는 객체를 넣었다. 이 객체의 `setOptions({fillColor, fillOpacity})`가 내부에서 `map.setFeatureState`를 호출하고, fill 레이어는 feature-state 값으로 색과 투명도를 정한다.
- InfoWindow는 `maplibregl.Popup`으로 바꿨다. `cells-fill` 레이어의 `mousemove`에서 격자 중심에 팝업을 띄우고, `mouseleave`에서 닫는다.
- 지도 관련 초기화와 키보드 핸들러는 `map.on("load")` 안에서 실행된다.

### foodscapeSeoul 전용

- 격자의 색과 불투명도는 feature `properties`(`fillColor`, `fillOpacity`)에 넣어 data-driven 스타일로 그린다.
- 호출되지 않던 원 그리기 코드(`dataInput_Circle`, `DrawCircle`)는 Google API를 참조하고 있어서 삭제했다.

### 공통

- `<meta charset="utf-8">`을 추가했다. chungnamcompany에는 BOM도 charset 선언도 없었다. 그래서 charset 헤더를 보내지 않는 로컬 서버(`python -m http.server`)에서는 한글이 깨졌다.

## 테스트

로컬 서버와 헤드리스 Chrome(Playwright)으로 세 페이지를 확인했다.

- Google로 나가는 요청이 0건이었다.
- 콘솔 오류는 favicon 404 하나뿐이었다.
- 격자 소스가 로드되고, 한글 라벨이 표시되었다(예: 서산시, 천안시, 한강).
- 업종 버튼과 교집합/합집합 토글로 격자가 다시 색칠되었다.
  - chungnam: 6,043개 중 2,484개
  - seoul: 8,473개 중 4,804개
- 격자에 마우스를 올리면 툴팁이 뜨고, 벗어나면 사라졌다.
- E 키로 투명도가 바뀌었고, 스페이스바로 초기화되었다.

## 참고

- Carto 베이스맵은 비상업적 용도로 키 없이 쓸 수 있고, 월간 사용량 한도가 있다. 한도가 문제가 되면 OpenFreeMap(`https://tiles.openfreemap.org/styles/positron` 등)으로 스타일 URL만 바꾸면 된다.
- 흰색 배경 사각형의 불투명도가 0.8이라서 베이스맵이 흐리게 보인다. 원래 디자인을 그대로 따른 것이다.
