<script>
import {onBeforeUnmount, onMounted, ref} from 'vue';
import 'ol/ol.css';
import Map from 'ol/Map';
import View from 'ol/View';
import { fromLonLat } from 'ol/proj'
import Feature from 'ol/Feature';
import { Point } from 'ol/geom';
import { Icon, Style } from "ol/style";
import { XYZ } from 'ol/source';
import { Tile as TileLayer, Vector as VectorLayer } from 'ol/layer';
import { Vector as VectorSource } from 'ol/source';
import { Overlay } from "ol";
import axios from "axios";
import { EventBus } from '../eventbus.js';
import { inAndOut } from 'ol/easing'
export default {
  name: 'MapComponent',
  setup() {
    let map;
    let baseLayer = [];
    let addLayer = [];
    let WarnPoint = [];
    let featureCoords = ref('')
    const points = [
      [118.759167,24.772778],
      [118.699167,24.772778],
      [118.718611,24.771944],
      [118.731111,24.756389],
      [118.730556,24.773056]
      //[118.7311111000,24.7843210000],
    ]
    const cameraSource = new VectorSource();
    const warningSource = new VectorSource();
    const currentpoint =[]

    onMounted(() => {
      initMap();
      EventBus.on('movepoint', (point) => {
        currentpoint.value=point
        movetopoint(currentpoint);
      });
    })
     onBeforeUnmount(() => {
       EventBus.off('movepoint');
        });

    function initMap(){
      fetchData();
      addCameraFeature(points);
      baseLayer = [
        //添加天地图web底图服务
        new TileLayer({
          source: new XYZ({
            url: 'http://120.37.123.14:58090/tiles/{z}/{x}/{y}.png'
          })
        }),
      ]
      let cameraLayer = new VectorLayer({
        name: '监控点',
        source: cameraSource
      })
      let warningLayer = new VectorLayer({
        name: '告警点状态',
        source: warningSource,
      })
      addLayer = [
        cameraLayer,
        warningLayer
      ]
      map = new Map({
        target: 'map',
        layers: [
            ...baseLayer,
            ...addLayer,
        ],
        view: new View({
          projection: "EPSG:4326",
          center: [118.708611,24.769444],
          zoom: 16,
          minzoom:8,
          maxzoom:17
        }),
      });  
      map.on('click', function(evt) {
        var feature = map.forEachFeatureAtPixel(evt.pixel, function(feature) {
          return feature;
        });

        if (feature) {
          // 如果点击到了要素，弹出窗口
          var coordinate = feature.getGeometry().getCoordinates();
          var content = '<p>经度: ' + coordinate[0] + '</p>' +
              '<p>纬度: ' + coordinate[1] + '</p>';
          featureCoords.value = coordinate[0] + ',' + coordinate[1]
          var popup = new Overlay({
            id: 0,
            element: document.getElementById('popup'),
            positioning: 'bottom-center',
            stopEvent: false,
            offset: [0, -50]
          });

          map.addOverlay(popup);
          popup.setPosition(coordinate);
          popup.getElement().style.display = 'block'
          document.getElementById('popup-content').innerHTML = content;
        }
      });
    }
    function movetopoint(newcenter){
      if (map && newcenter) {
        const center=newcenter.value
        console.log('newcenter',center)
        //map.getView().setCenter(fromLonLat(newCenter));
// 将经纬度转换为地图视图所使用的投影坐标
        const currentZoom = map.getView().getZoom();
// 缩小地图到某个级别（比如缩小到当前级别的一半）
        map.getView().animate({
          zoom: currentZoom - 1,
          duration: 800,
          easing: inAndOut
        }, function() {
          // 第一个动画完成后，执行移动到目标点的动画
          map.getView().animate({
            center: center,
            duration: 500,
            easing: inAndOut
          }, function() {
            // 移动到目标点后，执行放大动画
            map.getView().animate({
              zoom: currentZoom, // 恢复到原来的缩放级别
              center: center, // 保持中心不变
              duration: 800,
              easing: inAndOut
            });
          });
        });
      }
    }
    function handleSelect(value){
      addLayer.forEach((item)=>{
        if(!value.includes(item.get("name"))){
          item.setVisible(false)
          return;
        }
        item.setVisible(true)
      })
    }
    //切换至影像图层
    function switchToImg() {
      map.getLayers().setAt(0, new TileLayer({
        source: new XYZ({
          url: 'https://t6.tianditu.gov.cn/img_w/wmts?SERVICE=WMTS&REQUEST=GetTile&VERSION=1.0.0&LAYER=img&STYLE=default&TILEMATRIXSET=w&FORMAT=tiles&TILECOL={x}&TILEROW={y}&TILEMATRIX={z}&tk=22d8b67fcba918adc0a566e96d1587b3'
        })
      }));
    }

    // 切换至矢量图层
    function switchToVec() {
      map.getLayers().setAt(0, new TileLayer({
        source: new XYZ({
          url: 'https://t6.tianditu.gov.cn/vec_w/wmts?SERVICE=WMTS&REQUEST=GetTile&VERSION=1.0.0&LAYER=vec&STYLE=default&TILEMATRIXSET=w&FORMAT=tiles&TILECOL={x}&TILEROW={y}&TILEMATRIX={z}&tk=22d8b67fcba918adc0a566e96d1587b3'
        })
      }));
    }

    function addCameraFeature(points) {
      const cameraStyle = new Style({
        image: new Icon({
          src: "/img/camera.png",
          anchor: [0.5, 0.5],
        })
      })
      points.forEach(coordinates => {
        const pointGeometry = new Point(fromLonLat(coordinates, 'EPSG:4326'));
        const pointFeature = new Feature({
          geometry: pointGeometry,
        })
        pointFeature.setStyle(cameraStyle)
        cameraSource.addFeature(pointFeature)
        console.log(1)
      })
    }

    async function fetchData() {
      const warningStyle = new Style({
        image: new Icon({
          src: '/img/warning.png',
          anchor: [0.5, 1],
        })
      })
      try {
        const response = await axios.get('http://8.148.10.46:3050/api/MapComp');
        WarnPoint.splice(0, WarnPoint.length, ...response.data);
        console.log('WarningPoint',WarnPoint);
        WarnPoint.forEach(point => {
          const iconFeature = new Feature({
            geometry: new Point(fromLonLat([point.cenlon, point.cenlat],'EPSG:4326'))
          });
          iconFeature.setStyle(warningStyle);
          warningSource.addFeature(iconFeature)
          console.log('WarningSource features:', warningSource.getFeatures());
        });
      } catch (error) {
        console.error('Error fetching data:', error);
      }
    }//获取告警点位并添加

    function openCameraPage(coords){
      window.open(`/land-eye?center=${coords}`, '_blank');
    }
    function closePopUp(){
      map.getOverlayById(0).setPosition(undefined)
    }

    return {
      switchToImg,
      switchToVec,
      handleSelect,
      closePopUp,
      openCameraPage,
      featureCoords,
    };
  }
}
</script>

<template>
  <div id="map">
    <div id="popup" class="ol-popup" >
<!--      <a href="#" id="popup-closer" class="ol-popup-closer"></a>-->
      <div id="popup-content"></div>
      <a-button class="button" @click="openCameraPage(featureCoords)"><span>进入摄像头</span></a-button>
      <a-button  class="button" @click="closePopUp">关闭</a-button>
    </div>
    <div class="btn-group">
      <a-button class="btn" @click="switchToImg">影像</a-button>
      <a-button class="btn" @click="switchToVec">矢量</a-button>
    </div>
    <div class="select">
      <a-select
          mode="multiple"
          :default-value="['监控点','告警点状态']"
          style="margin-left:5px;width: 200px"
          placeholder="地图要素筛选"
          @change="handleSelect"
      >
        <a-select-option value="监控点">
          监控点
        </a-select-option>
        <a-select-option value="地块边界">
          地块边界
        </a-select-option>
        <a-select-option value="告警点状态">
          告警点状态
        </a-select-option>
        <a-select-option value="告警点级别">
          告警点级别
        </a-select-option>
      </a-select>
    </div>
    <div id="legend">
      <h2 class="legend-title">图例</h2>
      <div class="legend-item">
        <img src="/img/camera.png" alt="">
        <h3>摄像头</h3>
      </div>
      <div class="legend-item">
        <img src="/img/warning.png" alt="">
        <h3>告警点位</h3>
      </div>
    </div>
  </div>
</template>

<style>
#map {
  width:100%;
  height: 100%;
  position: relative;
}
#map .ol-zoom {
  display: flex;
  top:1vh;
  right: 18vh;
  background-color: rgba(255,255,255,0);
}
#map .ol-zoom .ol-zoom-in {
  position: absolute;
  right: 4vh;
  height: 6vh;
  width: 6vh;
  border-radius: 1vh;
}
#map .ol-zoom .ol-zoom-out {
  position: absolute;
  right: 12vh;
  height: 6vh;
  width: 6vh;
  border-radius: 1vh;
}
.btn-group {
  top:1vh;
  position: absolute;
  right: 2vh;
  z-index: 1000;
}
.btn {
  height: 6.5vh;
  width: auto;
  margin-right: 2vh;
  cursor: pointer;
  padding: 1vh 1vh;
  background-color: #3ea164;
  color: #e1e0e0;
  border: none;
  border-radius: 1vh;
  font-weight: bold;
  font-size: 2vh;
}
.btn:hover {
  background-color: #cdeec2;
  color: #676666 !important;
}
#legend {
  height: 35vh;
  width: 25vh;
  background-color: #c6debd;
  position: absolute;
  bottom: 0;
  right: 0;
  opacity: 0.8;
  z-index: 1000;
  flex-direction: column;
  border-radius: 1vh;
}
.legend-title {
  color:#1c1c1c;
  font-weight:bold;
  font-size: 3.2vh;
  margin-top: 2vh;
  padding-left: 10vh;
}
.legend-item {
  display: flex;
  align-items: center;
  padding-left: 2vh;
  margin-bottom: 1vh;
  img {
    margin-right: 1vh;
  }
  h3 {
    font-size: 2.5vh;
    font-weight: bold;
    color: #1c1c1c;
  }
}
.select {
  top: 2%;
  display: flex;
  left: 1%;
  position: absolute;
  width:50vh;
  z-index: 1000;
}
.ol-popup {
  position: absolute;
  box-shadow: 0 1px 4px rgba(0,0,0,0.2);
  padding: 15px;
  font-weight: bold;
  color: #4f4e4e;
  background-color: #e3e1d5;
  border-radius: 1vh;
  border: 1px solid #cccccc;
  bottom: -20px;
  left: -50px;
  min-width: 280px;
  display: none;
}

.ol-popup:after,
.ol-popup:before {
  top: 100%;
  border: solid transparent;
  content: " ";
  height: 0;
  width: 0;
  position: absolute;
  pointer-events: none;
}

.ol-popup:after {
  border-top-color: white;
  border-width: 10px;
  left: 48px;
  margin-left: -10px;
}

.ol-popup:before {
  border-top-color: #cccccc;
  border-width: 11px;
  left: 48px;
  margin-left: -11px;
}
#popup-content {
  text-align: center;
}
.button {
  margin-left: 28px;
  border:none;
  font-weight: bold;
  color:#4f4e4e;
}
.button span:hover {
   color:#ccc9c9!important
 }
</style>
