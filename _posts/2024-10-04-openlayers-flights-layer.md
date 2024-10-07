---
layout: post
date: 2024-10-04
title: "지도상에 항공 경로(원호) 표현하기"
published: true
lang: ko
excerpt: Openlayers와 turf 라이브러리로 래스터맵 위에 arc feature 표시
tags: gis
author: seounghyun
---

## 인구이동 차트 표현
진행중인 프로젝트에서 인구이동을 표현하기 위해 [highcharts][highcharts link] 라이브러리에서 지원하는 mapchart를 사용했습니다. mapchart는 지도상에 여러가지 데이터를 다양한 형태로 표현 가능하게 도와주는 라이브러리 포맷입니다. 해당 차트는 행정동에서 서울특별시로 유입되는 인구를 유입량, 지역, 시간대별로 표시하는 시각화 파트였습니다.  

![alt]({{ "/assets/2024-10-04-openlayers-flights-layer/2024-10-04-openlayers-flights-layer-01.png" | absolute_url}}){: .center-image } 

하지만 문제가 있었는데, 해당 차트에서 마우스 줌 인터랙션을 할 경우 지도가 버벅거리며 느리게 움직이는 현상이 발생했습니다. 기본적으로 지도는 행정동의 읍면동 단위까지 표현되어 있고, 모든 데이터가 존재하는 시군구에서 이동하는 feature를 전부 표현해서 그리기 때문에 vector 방식으로 표현되는 차트가 렌더링하는데 시간이 걸려 지연이 발생하는 것으로 추측했습니다.  

처음에는 highcharts의 옵션을 사용하거나, 계산을 위한 좌표계의 GeoJSON을 TopoJSON으로 변환하는등 여러 방법을 찾아봤지만, 제공하는 라이브러리의 데모에서도 똑같은 버벅임 현상이 발생하는걸 확인하고 자체 성능의 문제라고 판단해서 다른 방법을 찾기로 했습니다.

## VectorImageLayer
일반적으로 지도상에 레이어를 표시할때 소스에서 가져온 좌표값을 vector로 계산해서 표시하는데, 이는 사용자의 화면이나 인터랙션에 따라 유동적으로 feature를 계산해서 표시하기 위함입니다. 하지만 단점이 있는데, feature의 갯수가 수만~수십만이 될 경우 성능에 따라 브라우저가 화면에 렌더링하기 위한 속도가 저하되고 최악의 경우 지도가 표시되지 않을 수도있습니다.  

그래서 필요에 따라 적절하게 vector, tile, image 방식으로 레이어를 표시합니다. feature의 갯수가 너무 많아서 지도가 움직일때마다 vector를 계산하는게 어렵다면, 해당 역할을 서비스 서버에 넘길수도 있습니다. 보통 배경에 표시하는 래스터 레이어는 geoserver같은 GIS Server에 레이어를 발행하고, WMS를 사용해서 해당 좌표에 맞는 tile image를 응답받아 사용하곤 합니다.  

highcharts의 mapchart에는 vector image를 지원하는 옵션이 없어서 해당 파트에 openlayers를 적용 해봤습니다. 다만 이번에는 별도의 geoserver를 구축해서 WMS를 사용하는게 아닌 객체 생성시 ol.layer.VectorImage 클래스를 사용했습니다. 

## 원호(arc) 표현
가장 먼저 생각난건 Openlayers의 Exmamples에 있는 Flight Animation이었습니다. 예전에 예제를 찾아다니다 본적이 있기 때문에, 해당 비행 경로의 원호를 그리는 코드를 참고하면 응용해서 인구이동을 표현할때 사용할 수 있을 것으로 생각했습니다.  

![alt]({{ "/assets/2024-10-04-openlayers-flights-layer/2024-10-04-openlayers-flights-layer-02.png" | absolute_url}}){: .center-image } 
[Openlayers fligt animation][openlayers link]
{: style="text-align: center; font-style: italic;"}

하지만 코드를 분석하니 예제는 Animation 효과를 위해 동적으로 Line을 그리는 동작에 더 중점이 있는 것으로 보였고 해당 코드에서 원호를 그리기 위한 코드만을 분리해서 응용하기에는 무리가 있어 보였습니다. 그래서 다른 방법을 검색해서 찾아보던중, 원호를 그리기 위한 계산 공식과 필요한 라이브러리를 찾게 되어 해당 방향으로 구현하기로 했습니다.

지도상에 두 점을 잇는 원호를 그리기 위한 방식은, 다음과 같습니다.  

1.두 점의 좌표(여기선 출발과 도착지점)와 좌표 사이의 길이를 구합니다.  
2.두 좌표의 방위각을 구합니다.  
3.방위각과 좌표간의 중간 지점을 이용해 원의 중점을 구합니다.  
4.중점과 한 좌표를 기준으로 반지름(r)을 구합니다.  
5.반지름과 중점, 방위각을 사용해 원호를 그립니다.  
{: style="text-align: center; font-style: italic;"}

글로 설명하니 복잡해 보이지만, 그림과 코드를 함께 보면 어렵지 않습니다.  

## turf.js
지리공간 분석을 위해 각종 함수를 지원하는 [turf.js][turf.js link] 라이브러리를 사용하겠습니다. turf.js는 GeoJSON을 사용하는 모듈식 JavaScript  라이브러리 입니다. 서버로 데이터를 전송할 필요가 없고, 사용하고 싶은 모듈만 선택할 수 있습니다. 만약 이런 라이브러리를 사용하지 않는다면... 두 점 사이의 거리, 방위각을 구하는 복잡한 공식을 삼각함수를 사용해서 직접 구현해야 하기 때문에 매우 골치 아파질 수 있습니다. 다행히 turf.js에서 필요한 계산들을 전부 미리 구현해서 제공하기 때문에, 해당 함수만 적절히 사용하면 됩니다.  

✅ 자세한 공식과 실제로 값을 계산하는 과정이 궁금하신 분은 아래 링크를 참고하시면 됩니다.  
[https://www.movable-type.co.uk/scripts/latlong.html][latlong link]
{: .notice--info}

```javascript
let fromCenterCoord = fromCenter.geometry.coordinates;
let toCenterCoord = toCenter.geometry.coordinates;
```
먼저 출발지와 목적지가 될 두 좌표를 세팅하겠습니다. 좌표는 두개의 실수값으로 이루어진 배열 형태여야 합니다.
```javascript
let line = turf.lineString([fromCenterCoord, toCenterCoord]);
let d = turf.distance(fromCenterCoord, toCenterCoord,{units:"kilometers"});
let pMid = turf.along(line, (d / 2));
```
다음으로 두 좌표를 지나는 선분을 구합니다. 두 좌표사이의 거리를 계산하고, 그 거리의 절반을 구해 선분의 중점을 구할 수 있습니다.

![alt]({{ "/assets/2024-10-04-openlayers-flights-layer/2024-10-04-openlayers-flights-layer-03.png" | absolute_url}}){: .center-image } 

```javascript
let lineBearing = turf.bearing(fromCenterCoord, toCenterCoord);
```
출발지를 기준으로 도착지의 방위각을 구합니다. -180도와 180도 사이의 10진수 각도로 나타낸 값입니다.

![alt]({{ "/assets/2024-10-04-openlayers-flights-layer/2024-10-04-openlayers-flights-layer-04.png" | absolute_url}}){: .center-image } 

```javascript
let centerPoint = turf.destination(pMid, (2 * d), (lineBearing - 90));
let r = turf.distance(centerPoint, turf.point(fromCenterCoord));
```
이제 원호를 포함한 원의 중점을 구할 수 있습니다. 선분에서 방위각의 부호에 따라 4분면의 좌표에 위치할 방향을 구하고 선분의 길이의 두배 거리에 위치한 점이 원호를 그릴 원의 중점 좌표 입니다. 그리고 두 점중 한곳과 중점의 거리가 원의 반지름이 됩니다.

![alt]({{ "/assets/2024-10-04-openlayers-flights-layer/2024-10-04-openlayers-flights-layer-05.png" | absolute_url}}){: .center-image } 

```javascript
let bear1 = turf.bearing(centerPoint, turf.point(fromCenterCoord));
let bear2 = turf.bearing(centerPoint, turf.point(toCenterCoord));
let arc = turf.lineArc(centerPoint, r, bear2, bear1, {steps: 256});
```
마지막으로 두 좌표와 원의 중점 좌표간의 방위각을 구하고, 중점좌표, 반지름, 각 방위각, step을 사용하여 lineArc 함수로 원호 객체를 얻을 수 있습니다. steps options은 원호를 몇개의 선분으로 나눌지 세팅하는 값이며, 값이 클 수록 원호가 부드럽게그려집니다.

![alt]({{ "/assets/2024-10-04-openlayers-flights-layer/2024-10-04-openlayers-flights-layer-06.png" | absolute_url}}){: .center-image } 

```javascript
let arcFeature = new ol.format.GeoJSON().readFeatures(arc, {
        featureProjection: 'EPSG:4326',
        dataProjection: 'EPSG:4326'
    });

arcFeature[0].setStyle(new ol.style.Style({
    stroke: new ol.style.Stroke({
        color: '#FF0000',
        width: 5,
        lineCap: 'butt'
    })
}));

const flightsSource = new ol.source.Vector({});
flightsSource.addFeature(arcFeature[0]);

const flightsLayer = new ol.layer.Vector({
    source: flightsSource,
    zIndex: 99
});

map.addLayer(flightsLayer);
```
lineArc 함수로 얻은 원호(GeoJSON)값을 openlayers feature로 만들어주겠습니다. style을 적용하고 layer에 추가해서 map에 표시하면 아래처럼 표출되는걸 확인할 수 있습니다. 또한 vectorImage로 구현되는 레이어이기때문에, 속도측면에서도 훨씬 빠른 모습을 보이고 있습니다.

![alt]({{ "/assets/2024-10-04-openlayers-flights-layer/2024-10-04-openlayers-flights-layer-07.png" | absolute_url}}){: .center-image } 

이제 해당 feature에 스타일을 추가하고(화살표로 꾸민다던가) 클릭이벤트, 호버이벤트등을 추가하면 원하는 기능을 구현할 수 있겠습니다. 

+ 추가로 출발지, 도착지 좌표는 읍면동 폴리곤의 중심으로 설정했는데, 이 또한 turf.js에서 미리 지원하는 기능으로 손쉽게 multipolygon의 중점좌표를 구할 수 있었습니다.
```javascript
let from = turf.polygon([emdPolygonFrom]);
let to = turf.polygon([emdPolygonTo]);
```
<details>
<summary>전체 스크립트 보기</summary>
<div markdown="1">

```html
<script type="module">
    const map = new ol.Map({
        target: 'map',
        layers: [
            new ol.layer.Tile({
                source: new ol.source.OSM(),
            }),
        ],
        view: new ol.View({
            projection: "EPSG:4326",
            center: [126.9784147,37.5666805],
            zoom: 13,
        }),
    });
    async function getJsonData(){
        const response = await fetch("http://localhost:8088/emd.geojson");
        const data = await response.json();
        return data;
    }

    const emdhjdjson = await getJsonData();

    let vectorSource = new ol.source.Vector({
        format: new ol.format.GeoJSON(),
        loader: function () {
            const features = vectorSource.getFormat().readFeatures(emdhjdjson);
            features.forEach((el) => {
                el.setStyle(new ol.style.Style({
                    fill: new ol.style.Fill({
                        color: "rgba(29,54,49,0.24)",
                    }),
                    stroke: new ol.style.Stroke({
                        color: "rgba(0,34,255,0.53)",
                        width: 2,
                    }),
                    text: new ol.style.Text({
                        text: el.values_.EMD_KOR_NM.toString(),
                        scale: 1.3,
                        stroke: new ol.style.Stroke({color: '#FFFFFF', width: 2})
                    })
                }))
            })
            vectorSource.addFeatures(features);
        }
    })
    let vectorLayer = new ol.layer.VectorImage({
        source: vectorSource,
    });
    map.addLayer(vectorLayer);

    let emdPolygonFrom = emdhjdjson.features[0].geometry.coordinates[0][0];
    let emdPolygonTo = emdhjdjson.features[1].geometry.coordinates[0][0];

    let from = turf.polygon([emdPolygonFrom]);
    let to = turf.polygon([emdPolygonTo]);

    let fromCenter = turf.centerOfMass(from);
    let toCenter = turf.centerOfMass(to);

    let fromCenterCoord = fromCenter.geometry.coordinates;
    let toCenterCoord = toCenter.geometry.coordinates;

    let line = turf.lineString([fromCenterCoord, toCenterCoord]);
    let d = turf.distance(fromCenterCoord, toCenterCoord, {units:"kilometers"});
    let pMid = turf.along(line, (d / 2));
    let lineBearing = turf.bearing(fromCenterCoord, toCenterCoord);

    let centerPoint = turf.destination(pMid, (2 * d), (lineBearing - 90));

    let r = turf.distance(centerPoint, turf.point(fromCenterCoord));

    let bear1 = turf.bearing(centerPoint, turf.point(fromCenterCoord));
    let bear2 = turf.bearing(centerPoint, turf.point(toCenterCoord));
    let arc = turf.lineArc(centerPoint, r, bear2, bear1, {steps: 256});

    let arcFeature = new ol.format.GeoJSON().readFeatures(arc, {
        featureProjection: 'EPSG:4326',
        dataProjection: 'EPSG:4326'
    });
    arcFeature[0].setStyle(new ol.style.Style({
        stroke: new ol.style.Stroke({
            color: '#FF0000',
            width: 5,
            lineCap: 'butt'
        })
    }));
    const flightsSource = new ol.source.Vector({});
    flightsSource.addFeature(arcFeature[0]);

    const flightsLayer = new ol.layer.Vector({
        source: flightsSource,
        zIndex: 99
    });

    map.addLayer(flightsLayer);
</script>
```
</div>
</details>

## Reference
[https://gis.stackexchange.com/questions/384729/create-arc-lines-in-openlayers-like-kepler-gl][reference link]  
[https://www.movable-type.co.uk/scripts/latlong.html][latlong link]  
[https://openlayers.org/en/latest/examples/flight-animation.html][openlayers link]  

[reference link]:https://gis.stackexchange.com/questions/384729/create-arc-lines-in-openlayers-like-kepler-gl
[highcharts link]:https://www.highcharts.com/
[openlayers link]:https://openlayers.org/en/latest/examples/flight-animation.html
[turf.js link]:https://turfjs.org/
[latlong link]:https://www.movable-type.co.uk/scripts/latlong.html