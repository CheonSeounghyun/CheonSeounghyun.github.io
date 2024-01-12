---
layout: post
date: 2023-12-13
title: "카카오 지도위에 WMS 레이어 표시"
published: true
lang: ko
excerpt: Kakao Maps API를 활용하여 WMS 레이어를 표출해 봅니다
tags: gis
author: seounghyun
---

## Kakao Maps API
Kakao Maps API는 카카오가 제공하는 지도 서비스에 관련된 다양한 기능을 개발자들이 활용할 수 있는 프로그래밍 인터페이스를 제공하고있습니다. 이 API를 사용해 지리적 데이터, 길찾기, 장소 검색, 경로 탐색 등과 같은 다양한 지리 정보를 효과적으로 통합하고 활용할 수 있습니다. 또한 카카오 맵스 API를 활용하여 자사 애플리케이션 또는 웹 서비스에 동적이고 풍부한 지도 기능을 추가하고, 사용자에게 편리하고 직관적인 지도 서비스를 제공할 수 있습니다.  
  
현재 카카오 맵스에서 제공하는 지도 유형은 ROADMAP(일반지도), SKYVIEW(스카이뷰), HYBRID(하이브리드), ROADVIEW(로드뷰), OVERLAY(레이블), TRAFFIC(교통정보), TERRAIN(지형도), BICYCLE(자전거), BICYCLE_HYBRID(스카이뷰 자전거), USE_DISTRICT(지적편집도) 총 10가지 입니다.  
  
이중 베이스타입으로 사용하는 ROADMAP, SKYVIEW, HYBIRD를 제외한 나머지 유형은 오버레이 타입으로 베이스타입 유형 위에 오버레이 타일을 올리거나 걷어내는 용도로 사용합니다.

## WMS (Web Map Service)
WMS(웹 맵 서비스)는 지리 정보 시스템(GIS)에서 사용하는 표준 프로토콜중 하나로, 지리적 데이터를 웹 상에서 효과적으로 공유하고 표현하기 위한 서비스입니다. WMS는 지리 정보를 이미지 형태로 제공하며, 클라이언트 어플리케이션에서는 이 이미지를 받아 지도 형태로 표시합니다. 사용자는 다양한 지리 정보를 시각적으로 확인할 수 있습니다. 또한 WMS는 여러 소스에서 제공되는 지리 정보를 표준화된 방식으로 통합하여 사용자에게 일관적인 지도 경험을 제공하는데 사용되고 있습니다. 이번 게시글에서 WMS를 활용하여 커스텀 레이어를 카카오 지도에 추가하고 표출시키는 방법을 설명하도록 하겠습니다.

✅ 아래 예제에서는 WMS Layer 연계 및 등록에 대한 설명은 생략하고 WMS 호출 소스가 준비되어 있다는 가정하에 설명하겠습니다.
{: .notice--info}

## Kakao Maps 기본 지도 화면
```html
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>카카오맵 테스트</title>
    <script type="text/javascript" src="//dapi.kakao.com/v2/maps/sdk.js?appkey=yourkey"></script>
    
    <style>
        body, html {height: 100%; margin: 0px;}
    </style>
</head>
<body>
    <div id="map" style="width:100%;height:100%;"></div>
</body>
    <script>
        var container = document.getElementById('map');
        var options = {
            center: new kakao.maps.LatLng(37.566535, 126.9779692),
            level: 8
        };

        var map = new kakao.maps.Map(container, options);
    </script>
</html>
```
먼저 카카오 기본 지도를 표시하는 화면을 하나 표시하겠습니다. kakao API dev 사이트에서 라이센스 key를 발급받아서 화면 전체에 기본 ROADMAP 레이어를 추가합니다.  

![alt]({{ "/assets/2023-12-13-kakao-map-wms-layer/2023-12-13-kakao-map-wms-layer-01.png" | absolute_url}}){: .center-image }

다음으로 카카오 지도 위에 올릴 WMS 레이어를 준비하겠습니다. 저는 읍면동 행정구역 표시 레이어를 준비했습니다. [대한민국 최신 행정구역][emd-source]에서 읍면동 데이터를 받아서 GeoServer에 쉐이프파일을 레이어로 올리도록 하겠습니다. PostGIS와 GeoServer를 연계해서 레이어를 발행하는 방법은 본 게시글에서는 생략하겠습니다. 

![alt]({{ "/assets/2023-12-13-kakao-map-wms-layer/2023-12-13-kakao-map-wms-layer-02.png" | absolute_url}}){: .center-image }

이제 해당 레이어를 카카오 지도 위에 올려보도록 하겠습니다. kakao.maps.MarkerImage생성자로 마커이미지를 생성하고, 마커이미지로 마커를 생성한 후에 지도에 추가합니다.

```javascript
// 마커 이미지 초기화
let imageSize = new kakao.maps.Size(container.clientWidth, container.clientHeight);
let centerlatlng = map.getCenter();
let bounds = map.getBounds();
let minXY = bounds.getSouthWest();
let maxXY = bounds.getNorthEast();
let minCoordinate = [minXY.getLng(),minXY.getLat()];
let maxCoordinate = [maxXY.getLng(),maxXY.getLat()];
let imageSrc = 'http://localhost:9090/geoserver/gis/wms?' +
    'SERVICE=WMS&VERSION=1.1.1&REQUEST=GetMap&FORMAT=image%2Fpng&TRANSPARENT=true&STYLES=' +
    '&LAYERS=emd' +
    '&SRS=EPSG%3A5181' +
    '&WIDTH=' + (imageSize.width) + '&HEIGHT=' + (imageSize.height) +
    '&BBOX=' + minCoordinate[0] + '%2C' + minCoordinate[1] + '%2C' + maxCoordinate[0] + ' %2C' + maxCoordinate[1];
let imageOption = {
    offset: new kakao.maps.Point(imageSize.width / 2, imageSize.height / 2)
};
let markerImage = new kakao.maps.MarkerImage(imageSrc, imageSize, imageOption);
```
마커 이미지의 크기를 계산하기 위해 컨테이너에서 width, height를 가져와서 kakao.maps.size 객체를 초기화합니다. GeoServer에 WMS 이미지를 요청하기 위한 파라미터를 세팅합니다. 현재 지도가 표시되고 있는 DOM 객체의 범위를 계산해서, 북동쪽 좌표 정보, 남서쪽 좌표 정보의 좌표계 값을 구하고 각각의 위도 경도값으로 요청할 이미지의 크기와 좌표정보를 전달합니다.

```javascript
// 마커 객체 추가
let markerPosition = new kakao.maps.LatLng(centerlatlng.getLat(), centerlatlng.getLng());
let marker = new kakao.maps.Marker({
    position: markerPosition,
    image: markerImage,
    opacity : 0.5
});

marker.setMap(map);
```
마커 객체를 생성하고 맵에 추가합니다. 배경지도와 읍면동 경계를 구분하기 위해 마커 이미지의 투명도를 조절하겠습니다.

![alt]({{ "/assets/2023-12-13-kakao-map-wms-layer/2023-12-13-kakao-map-wms-layer-03.png" | absolute_url}}){: .center-image }

지도위에 읍면동레이어가 잘 올라간 모습입니다. 그런데 두가지 문제가 있습니다. 지도를 zoom 해보면...

![alt]({{ "/assets/2023-12-13-kakao-map-wms-layer/2023-12-13-kakao-map-wms-layer-04.png" | absolute_url}}){: .center-image }

지도의 위치와 마커이미지가 서로 맞지 않습니다. 현재 마커 이미지는 최초 1번 지도가 로드되면서 요청하므로, 초기화면의 사각영역 정보와 크기를 기준으로 갖고 있는 이미지이기 때문입니다. 또한 지도와 WMS이미지의 좌표계가 맞지 않습니다. 지도가 이동되거나, 줌이벤트가 발생할 때마다 현재 지도기준으로 계산하여 WMS로 마커 이미지를 요청해서 해당 마커 이미지의 정보를 갱신하도록 수정하겠습니다.

✅ 카카오 지도 API는 기본적으로 EPSG:5181 좌표계를 사용하지만, map.getBounds()에서 리턴하는 LatLngBounds값은 WGS84(EPSG:4326) 좌표계의 사각영역 정보입니다. 만약 WMS에 등록한 레이어의 좌표계가 다르다면 openLayers의 proj 라이브러리나, 카카오 지도 API의 transCoord 함수를 사용하여 좌표계를 변환해야 합니다.
{: .notice--info}

- WMS Layer 좌표계 : EPSG 5181
- Kakao Maps 좌표계 : EPSG 5181
- WGS84 : EPSG 4326  

WMS에 5181 좌표계를 기준으로 이미지를 요청해야 하기때문에 LatLngBounds값의 좌표를 OpenLayers의 proj라이브러리를 사용해서 5181 좌표계로 변환하겠습니다.

```html
<!-- header에 추가 -->
<script src="https://cdn.jsdelivr.net/npm/ol@v8.2.0/dist/ol.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/proj4js/2.5.0/proj4.js"></script>
```
```javascript
//스크립트에 추가
proj4.defs("EPSG:5181", "+proj=tmerc +lat_0=38 +lon_0=127 +k=1 +x_0=200000 +y_0=500000 +ellps=GRS80 +units=m +no_defs");
ol.proj.proj4.register(proj4);
```
마커 이미지에서 WMS에 요청하는 좌표 영역을 전달하는 코드를 아래처럼 변환해서 전달하도록 하겠습니다.
```javascript
//let minCoordinate = [minXY.getLng(),minXY.getLat()];
//let maxCoordinate = [maxXY.getLng(),maxXY.getLat()];
let minCoordinate = ol.proj.transform([minXY.getLng(),minXY.getLat()],"EPSG:4326","EPSG:5181");
let maxCoordinate = ol.proj.transform([maxXY.getLng(),maxXY.getLat()],"EPSG:4326","EPSG:5181");
```
![alt]({{ "/assets/2023-12-13-kakao-map-wms-layer/2023-12-13-kakao-map-wms-layer-05.png" | absolute_url}}){: .center-image }

이제 정상적으로 좌표에 맞게 지도와 WMS이미지가 표시되는 모습입니다. 마지막으로 지도 객체에 이벤트를 추가해서 줌레벨이나 중심좌표가 변경될때 마커 이미지를 다시 요청해서 뿌리는 방식으로 코드를 변경해보겠습니다.

```javascript
kakao.maps.event.addListener(map, "idle", function(){
    marker.setMap(null);
    marker.setImage(getMarkerImage());
    marker.setPosition(map.getCenter());
    marker.setMap(map);
});

function getMarkerImage(){
    let imageSize = new kakao.maps.Size(container.clientWidth, container.clientHeight);
    ...
    return markerImage;
}
```
마커 이미지 정의 부분을 마커 이미지를 리턴하는 함수로 변경한뒤 `idle` 이벤트로 map 객체에 마커를 제거하고 이미지를 바꾸고 위치를 바꾼뒤에 다시 추가하는 로직을 등록합니다.

![alt]({{ "/assets/2023-12-13-kakao-map-wms-layer/2023-12-13-kakao-map-wms-layer-06.webp" | absolute_url}}){: .center-image }

지도가 이동될때마다 WMS 이미지를 서버에 요청해서 마커로 표시하는 모습을 확인할 수 있습니다.  

<details>
<summary>전체 코드 보기</summary>
<div markdown="1">

```html
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>카카오맵 테스트</title>
    <script src="https://cdn.jsdelivr.net/npm/ol@v8.2.0/dist/ol.js"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/proj4js/2.5.0/proj4.js"></script>

    <script type="text/javascript" src="//dapi.kakao.com/v2/maps/sdk.js?appkey=yourkey"></script>
    <style>
        body, html {height: 100%; margin: 0px;}
    </style>
</head>
<body>
    <div id="map" style="width:100%;height:100%;"></div>
</body>
    <script>
        //*중부원점(GRS80)-falseY:50000: 카카오 지도에서 사용중인 좌표계
        proj4.defs("EPSG:5181", "+proj=tmerc +lat_0=38 +lon_0=127 +k=1 +x_0=200000 +y_0=500000 +ellps=GRS80 +units=m +no_defs");
        ol.proj.proj4.register(proj4);

        let container = document.getElementById('map');
        let options = {
            center: new kakao.maps.LatLng(37.566535, 126.9779692),
            level: 8
        };

        let map = new kakao.maps.Map(container, options);

        // 마커 객체 추가
        let markerPosition = new kakao.maps.LatLng(map.getCenter().getLat(), map.getCenter().getLng());
        let marker = new kakao.maps.Marker({
            position: markerPosition,
            image: getMarkerImage(),
            opacity: 0.5
        });
        marker.setMap(map);

        kakao.maps.event.addListener(map, "idle", function(){
            marker.setMap(null);
            marker.setImage(getMarkerImage());
            marker.setPosition(map.getCenter());
            marker.setMap(map);
        });

        function getMarkerImage(){
            // 마커 이미지 소스
            let imageSize = new kakao.maps.Size(container.clientWidth, container.clientHeight);
            let bounds = map.getBounds();
            let minXY = bounds.getSouthWest();
            let maxXY = bounds.getNorthEast();
            let minCoordinate = ol.proj.transform([minXY.getLng(),minXY.getLat()],"EPSG:4326","EPSG:5181");
            let maxCoordinate = ol.proj.transform([maxXY.getLng(),maxXY.getLat()],"EPSG:4326","EPSG:5181");
            let imageSrc = 'http://localhost:9090/geoserver/gis/wms?' +
                'SERVICE=WMS&VERSION=1.1.1&REQUEST=GetMap&FORMAT=image%2Fpng&TRANSPARENT=true&STYLES=' +
                '&LAYERS=emd' +
                '&SRS=EPSG%3A5181' +
                '&WIDTH=' + (imageSize.width) + '&HEIGHT=' + (imageSize.height) +
                '&BBOX=' + minCoordinate[0] + '%2C' + minCoordinate[1] + '%2C' + maxCoordinate[0] + ' %2C' + maxCoordinate[1];
            let imageOption = {
                offset: new kakao.maps.Point(imageSize.width / 2, imageSize.height / 2)
            };
            let markerImage = new kakao.maps.MarkerImage(imageSrc, imageSize, imageOption);

            return markerImage;
        }
    </script>
</html>
```
</div>
</details>

[emd-source]: http://www.gisdeveloper.co.kr/?p=2332