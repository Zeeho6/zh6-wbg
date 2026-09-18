<template>
  <div class="top-nav">
    <div class="city-choose">{{ city }}</div>
    <el-input v-model="searchCity" placeholder="请输出入城市名称" style="width:200px" size="large" :prefix-icon="Search"
      @keyup.enter="handleSearchCity">
    </el-input>
    <el-select v-model="drawType" placeholder="请选择绘画类型" style="width:200px" size="large" @change="handleDrawTypeChange">
      <el-option v-for="item in options" :key="item.value" :label="item.label" :value="item.value"></el-option>
    </el-select>
    <el-select v-model="iconType" placeholder="请选择标记类型" style="width:200px" size="large" @change="handleIconTypeChange">
      <el-option v-for="item in iconOptions" :key="item.value" :label="item.label" :value="item.value"></el-option>
    </el-select>
  </div>
  <div id="map-container"></div>
  <div id="popup" v-html="popupContent"></div>
  <div id="BaseMapSwitch">
    <el-button-group>
      <el-button type="plain" @click="swtsl">矢量底图</el-button>
      <el-button type="plain" @click="swtyx">遥感影像</el-button>
    </el-button-group>
  </div>
</template>

<script setup>
import { Search } from '@element-plus/icons-vue';
import { ref, onMounted } from 'vue';
import Map from 'ol/Map.js';
import View from 'ol/View.js';
import TileLayer from 'ol/layer/Tile.js';
import VectorLayer from 'ol/layer/Vector.js';
import VectorSource from 'ol/source/Vector.js';
import GeoJSON from 'ol/format/GeoJSON.js';
import Style from 'ol/style/Style.js';
import Fill from 'ol/style/Fill.js';
import Stroke from 'ol/style/Stroke.js';
import Draw from 'ol/interaction/Draw.js';
import XYZ from 'ol/source/XYZ.js';
import Overlay from 'ol/Overlay';
import Feature from 'ol/Feature';
import { Point } from 'ol/geom';
import Icon from 'ol/style/Icon';
import { Cluster} from 'ol/source';
import Text from 'ol/style/Text';
import CircleStyle from 'ol/style/Circle.js';
const city = ref('')
const searchCity = ref('');
const adcode = ref('');
const cityLocation = ref([]);
const drawType = ref('');
const iconType = ref('');
const popupContent = ref('');
let mapClickHandler=null;
const options = [
  { label: '直线', value: 'line' },
  { label: '圆形', value: 'circle' },
  { label: '多边形', value: 'polygon' },
  { label: '自由画笔', value: 'freehand' },
  { label: '退出绘图', value: 'exit-draw'  },
  { label: '清除绘图', value: 'clear-draw' },];
const iconOptions = [
  { label: '充电站', value: 'charging-station' },
  { label: '公交站', value: 'bus-station' },
  { label: '停车场', value: 'parking-lot' },
  { label: '退出标记', value: 'exit-mark' },
  { label: '清除标记' , value:'clear-mark'}
];
const iconStyles={
  'charging-station':'/新能源充电站.png',
  'bus-station':'/公交站.png',
  'parking-lot':'/停车场.png',
};
//加载地图
let map, popup, cityLayer, drawLayer, iconLayer,iconSource, draw;
let sldtLayer,slzjLayer,ygyxLayer,yxzjLayer;
const gaodeKey = import.meta.env.VITE_AMAP_KEY;
const tiandituKey=import.meta.env.VITE_TIANDITU_KEY;
onMounted(async () => {
sldtLayer = new TileLayer({
        source: new XYZ({
          url: `https://t0.tianditu.gov.cn/vec_w/wmts?SERVICE=WMTS&REQUEST=GetTile&VERSION=1.0.0&LAYER=vec&STYLE=default&TILEMATRIXSET=w&FORMAT=tiles&TILEMATRIX={z}&TILEROW={y}&TILECOL={x}&tk=${tiandituKey}`,
        }),
      });
slzjLayer = new TileLayer({
        source: new XYZ({
          url: `https://t0.tianditu.gov.cn/cva_w/wmts?SERVICE=WMTS&REQUEST=GetTile&VERSION=1.0.0&LAYER=cva&STYLE=default&TILEMATRIXSET=w&FORMAT=tiles&TILEMATRIX={z}&TILEROW={y}&TILECOL={x}&tk=${tiandituKey}`,
        }),
      });
ygyxLayer = new TileLayer({
        source: new XYZ({
          url: `https://t0.tianditu.gov.cn/img_w/wmts?SERVICE=WMTS&REQUEST=GetTile&VERSION=1.0.0&LAYER=img&STYLE=default&TILEMATRIXSET=w&FORMAT=tiles&TILEMATRIX={z}&TILEROW={y}&TILECOL={x}&tk=${tiandituKey}`,
        }),
      });
yxzjLayer = new TileLayer({
        source: new XYZ({
          url: `https://t0.tianditu.gov.cn/cia_w/wmts?SERVICE=WMTS&REQUEST=GetTile&VERSION=1.0.0&LAYER=cia&STYLE=default&TILEMATRIXSET=w&FORMAT=tiles&TILEMATRIX={z}&TILEROW={y}&TILECOL={x}&tk=${tiandituKey}`,
        }),
      });
  map = new Map({
    target: 'map-container',
    view: new View({
      projection: 'EPSG:4326',
      center: [120.15507, 30.27415],
      zoom: 4,
    }),
    layers: [ygyxLayer,yxzjLayer,sldtLayer,slzjLayer],
  });
  ygyxLayer.setVisible(false);
  yxzjLayer.setVisible(false);
  popup = new Overlay({
    element: document.getElementById('popup'),
  });
  map.addOverlay(popup);
  drawLayer = new VectorLayer({
    source: new VectorSource(),
    style: new Style({
      fill: new Fill({
        color: 'rgba(255,0,0,0.5)',
      }),
      stroke: new Stroke({
        color: 'black',
        width: 1,
      }),
    }),
  }); 
  map.addLayer(drawLayer);
  iconSource = new VectorSource();
  const clusterSource = new Cluster({
    distance:40,
    source:iconSource,
  });
  function clusterStyleFunction(feature) {
  const features = feature.get('features'); // 聚合的原始要素数组
  const size = features.length;

  if (size > 1) {
    // 聚合点样式：橙色圆圈 + 数量文字
    return new Style({
      image: new CircleStyle({
        radius: 15,
        fill: new Fill({ color: 'rgba(255, 153, 0, 0.8)' }),
        stroke: new Stroke({ color: '#cc3300', width: 2 }),
      }),
      text: new Text({
        text: size.toString(),
        fill: new Fill({ color: '#fff' }),
        font: '12px sans-serif',
      }),
    });
  }

  // 单个要素：返回原始要素的样式（即你设置的 Icon）
  const originalFeature = features[0];
  // 如果原始要素有样式，直接返回；否则可返回默认样式
  return originalFeature.getStyle() || new Style({
    image: new CircleStyle({
      radius: 8,
      fill: new Fill({ color: 'blue' }),
    }),
  });
};
  iconLayer = new VectorLayer({
    source:clusterSource,
    style:clusterStyleFunction,
  });
  map.addLayer(iconLayer);
  const temp = await fetch(`https://restapi.amap.com/v3/ip?key=${gaodeKey}`);
  const res = await temp.json();
  city.value = res.city;
  adcode.value = res.adcode;
  const temp2 = await fetch(`https://restapi.amap.com/v3/geocode/geo?address=${city.value}&key=${gaodeKey}`);
  const res2 = await temp2.json();
  cityLocation.value = res2.geocodes[0].location.split(',').map(Number);
    cityLayer = new VectorLayer({
      source: new VectorSource({
        url: `https://geo.datav.aliyun.com/areas_v3/bound/${adcode.value}.json`,
        format: new GeoJSON(),
      }),
      style: new Style({
        fill: new Fill({
          color: 'rgba(255,0,0,0.5)',
        }),
        stroke: new Stroke({
          color: 'black',
          width: 1,
        }),
      }),
    });
    map.getView().animate({
      center: cityLocation.value,
      zoom: 9,
      duration: 1000,
    },()=>{
      map.addLayer(cityLayer);
    });
  map.getView().on("change:resolution", function () {
    let zoom = this.getZoom();
    if (zoom > 10) {
      cityLayer.setVisible(false);
      drawLayer.setVisible(false);
    } else {
      cityLayer.setVisible(true);
      drawLayer.setVisible(true);
    }
  });
  let hoverTimer=null;
  map.on('pointermove', function (e) {
    if(e.dragging) return;
    if(drawType.value && iconType.value !== 'clear-mark'&&iconType.value !=='exit-mark') return;
    clearTimeout(hoverTimer);
    const feature = map.getFeaturesAtPixel(e.pixel,{
      layerFilter: (layer) => layer === cityLayer,
    });
    if (feature.length > 0 && feature[0].get('name')) {
      const center = feature[0].get('center');
      const name = feature[0].get('name');
      hoverTimer=setTimeout(()=>{
        popupContent.value = `<p>当前城市位置为：${name}</p>
      <p>当前经度：${center[0]}</p>
      <p>当前纬度：${center[1]}</p>`
      popup.setPosition(center);
      },500);
    } else {
      popup.setPosition(undefined);
    }
  }
  );
  map.on('click',function(e){
    const iconFeatures = map.getFeaturesAtPixel(e.pixel,{
      layerFilter: (layer) => layer === iconLayer,
    });
    if(iconFeatures.length===0) return;
    const feature=iconFeatures[0];
    const originals = feature.get('features') || []
    if(originals.length>1){
      map.getView().animate({
      center: feature.getGeometry().getCoordinates(),
      zoom: map.getView().getZoom() + 2,
      duration: 1000,
    });
    }
  });
});
async function handleSearchCity() {
  map.removeLayer(cityLayer);
  const temp = await fetch(`https://restapi.amap.com/v3/geocode/geo?address=${searchCity.value}&key=${gaodeKey}`);
  const res = await temp.json();
  if (res.status === "0") {
    searchCity.value = '';
    alert('未找到该城市，请重新输入');
    return;
  }
  city.value = res.geocodes[0].city;
  adcode.value = res.geocodes[0].adcode;
  cityLocation.value = res.geocodes[0].location.split(',').map(Number);
  cityLayer = new VectorLayer({
    source: new VectorSource({
      url: `https://geo.datav.aliyun.com/areas_v3/bound/${adcode.value}.json`,
      format: new GeoJSON(),
    }),
    style: new Style({
      fill: new Fill({
        color: 'rgba(255,0,0,0.5)',
      }),
      stroke: new Stroke({
        color: 'black',
        width: 1,
      }),
    }),
  });
  map.getView().animate({
    center: cityLocation.value,
    zoom: 9,
    duration: 1000,
  },()=>{
    map.addLayer(cityLayer);
  });
};
function handleDrawTypeChange() {
  if(iconType.value !=='exit-mark'&&iconType.value!=='clear-mark'){
    iconType.value = '';
    if (mapClickHandler) {
      map.un('click', mapClickHandler);
      mapClickHandler = null;
    };
  };
  if (draw) {
    map.removeInteraction(draw);
    draw = null;
  };
  if (drawType.value === 'exit-draw') {
    drawType.value = '';
    return;
  }
  if (drawType.value === 'clear-draw') {
    drawLayer.getSource().clear();
    drawType.value = "";
  } else {
    map.removeInteraction(draw);
    if (drawType.value === 'line') {
      draw = new Draw({
        type: "LineString",
        source: drawLayer.getSource(),
      });
    } else if (drawType.value === "circle") {
      draw = new Draw({
        type: "Circle",
        source: drawLayer.getSource(),
      });
    } else if (drawType.value === "polygon") {
      draw = new Draw({
        type: "Polygon",
        source: drawLayer.getSource(),
      });
    } else {
      draw = new Draw({
        type: "LineString",
        source: drawLayer.getSource(),
        freehand: true,
      });
    };
    map.addInteraction(draw);
  };
};
function handleMapClick(e){
  const iconFeatures = map.getFeaturesAtPixel(e.pixel, {
    layerFilter: (layer) => layer === iconLayer,
  });
  if (iconFeatures.length > 0) return;
  const coordinate=e.coordinate;
  const iconStyle=iconStyles[iconType.value];
  if(!iconStyle)return;
  const feature=new Feature({
    geometry:new Point(coordinate),
    name:iconType.value,
  });
  feature.setStyle(
    new Style({
      image:new Icon({
        src:iconStyle,
        scale:0.1,
      }),
    }),
  );
  iconSource.addFeature(feature);
};
function handleIconTypeChange(){
  if (iconType.value === 'exit-mark') {
    iconType.value = '';
    popup.setPosition(undefined);
    return;
  };
  if(iconType.value==='clear-mark'){
    iconSource.clear();
    iconType.value='';
    return;
  }else{
    if (drawType.value!=='clear-draw'&&drawType.value!=='exit-draw'){
    map.removeInteraction(draw);
    drawType.value = '';
    }
    if (mapClickHandler) {
    map.un('click', mapClickHandler);
    mapClickHandler = null;
  }
  if (iconType.value && iconType.value !== 'exit-mark') {
    mapClickHandler = handleMapClick;
    map.on('click', mapClickHandler);
  }
}
};
function swtsl(){
  sldtLayer.setVisible(true);
  ygyxLayer.setVisible(false);
  slzjLayer.setVisible(true);
  yxzjLayer.setVisible(false);
};
function swtyx(){
  ygyxLayer.setVisible(true);
  sldtLayer.setVisible(false);
  yxzjLayer.setVisible(true);
  slzjLayer.setVisible(false);
};
</script>

<style lang="scss" scoped>
.top-nav {
  height: 80px;
  background-color: blue;
  position: fixed;
  width: 100%;
  display: flex;
  align-items: center;
  gap: 100px;

  .city-choose {
    margin-inline: 80px;
    color: white;
  }
}

;

#map-container {
  position: absolute;
  top: 80px;
  width: 100%;
  height: calc(100% - 80px);
}

;

#popup {
  width: 200px;
  height: min-content;
  padding: 10px;
  border-radius: 5px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.15);
  background-color: pink;
  pointer-events: none;
}

;
#BaseMapSwitch{
  position: fixed;
  bottom:0px;
  left:0px;
}
</style>