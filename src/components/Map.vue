<template>
<div class = "main">    
    <div id="map-container" class="map-container"></div>
    <!-- <LayerSwitchControl /> -->
    <!-- 底图切换控件 容器 需要单独提取为LayerSwitchControl, 直接提取有bug-->
   <div id='layers' class="ol-control notPhone" style="bottom:2.5em;background:white;left:0.5em;border:1px solid #969696;opacity:0.8">
      <img src="../assets/BasemapInactive.png" id="bming" style="margin:1px">
      <div id='lyscontent' style="margin:6px;display:none;float:left;height:250px">
        <div style="float:left;height:25px">
          <span style="background:#00688b;font-size:14px;border-radius:7px;padding:2px 6px 3px;color:white">底图切换</span>
        </div>
        <div id="pnlBasemaps" style="clear:both;margin-top:5px">
          <div class="bmapGem" id="emptyLayer">
            <img src="../assets/empty.png" class="bmImg">
            <br>
            Empty
          </div>
          <div class="bmapGem" id="osmLayer">
            <img src="../assets/esriStreet.png" class="bmImg">
            <br>
            Open Street Map
          </div>
          <div class="bmapGem" id="satelliteLayer_2020">
            <img src="../assets/aerials2010.png" class="bmImg">
            <br>
            Google
          </div>
          <div class="bmapGem" id="hybridLayer_2020" style="clear:both">
            <img src="../assets/aerials2010.png" class="bmImg">
            <br>
            Google (hybrid)
          </div>
          <div class="bmapGem" id="esri-satellite">
            <img src="../assets/aerials2010.png" class="bmImg">
            <br>
            Esri-satellite
          </div>
          <div class="bmapGem" id="stamen">
            <img src="../assets/esriGray.png" class="bmImg">
            <br>
            Stamen
          </div>
        </div>
        <div style="float:left;height:25px">
          <span style="background:#00688b;font-size:14px;border-radius:7px;padding:2px 6px 3px;color:white">加载Overlay</span>
        </div>
        <div id = "overlays" style="clear:both;margin-top:5px">
          <div class="overlayGem" id="emptyLayer">
            <img src="../assets/empty.png" class="overlayImg">
            <br>
            Empty
          </div>
          <div class="overlayGem" id="geoJson">
            <img src="../assets/esriStreet.png" class="geoJson">
            <br>
            geoJson
          </div>
        </div>
        </div>
      </div>
  </div>
</template>
<script>
import 'ol/ol.css';
import Map from 'ol/Map';
import View from 'ol/View';
import {Circle as CircleStyle, Fill, Stroke, Style} from 'ol/style';
import {Polygon} from 'ol/geom';
import {OSM, Vector as VectorSource, ImageWMS} from 'ol/source';
import {Tile as TileLayer, Vector as VectorLayer,Image as ImageTile} from 'ol/layer';
import {sync} from "vuex-pathify";
import Feature from 'ol/Feature';
import '@/styles/ol3-layerswitcher.css';
import {defaults as defaultControls, ScaleLine} from 'ol/control';
// 导入自定义的控制模块, 必须加 {}, 不然会报错
import {CreateDrawControl} from '@/util/customcontrols/createdrawcontrol'
import {BackDrawControl} from '@/util/customcontrols/backdrawcontrol'
import {ClearDrawControl} from '@/util/customcontrols/cleardrawcontrol'
import {LayersControl } from '../util/customcontrols/layerscontrol';
import XYZ from 'ol/source/XYZ'
import TileArcGISRest from 'ol/source/TileArcGISRest'
import Stamen from 'ol/source/Stamen'
// import { truncate } from 'fs';

//基于变量在外部, 监听函数内存在访问不到的问题(javascript基础问题), 变量被统一定义在data域中
export default {
  name: 'Map',
  data() {
    return {
      /**
       * Openlayers地图对象           ，。
       * @type {Object}
       */
      map: null,
      baseMap: {
        baseMap: null
      },
      overlay: {
        overlay: null
      },
      layers : null,
      isSelectGeojson: false,
      /**
       * 测试用的Geojson roi数组
       * @type {Array}
       */
       geoJsonArray: [
        {"type" : "Feature","properties": "","geometry" : {"type": "Polygon", "coordinates":  [11591141.978821806609631, 3546469.098744962830096 , 11577227.042500456795096, 3546469.098744962830096 , 11577227.042500456795096, 3557221.684605775866657 ,11591141.978821806609631, 3557221.684605775866657 ,11591141.978821806609631, 3546469.098744962830096 ] ,"realcoordinates": ""}},
        {"type" : "Feature","properties": "","geometry" : {"type": "Polygon", "coordinates":  [ 11591141.978821806609631, 3557221.684605775866657 , 11577227.042500456795096, 3557221.684605775866657 , 11577227.042500456795096, 3567983.455974561162293 , 11591141.978821806609631, 3567983.455974561162293 , 11591141.978821806609631, 3557221.684605775866657 ] ,"realcoordinates": ""}}
       ],
            /**
       * 用户绘制roi的面积，用字典解决无法在其他地方修改变量值的问题, roi:region of interest
      */
      roiArea: {
        roiArea : ''
      },
      /**
       * 一维数组, 依次存放用户绘制区域点的经纬度，保存用户点信息需要用到这个数组
       */
      coordinateArray: {
        coordinateArray : []
      },
      /**
       * Openlayers绘图对象           ，。
       */
      draw: {
        draw : null
      },
      /**
       * 用户绘制的region of interest polygon数组
       */
      userDrawnPolygons : {
        userDrawnPolygons : []
      },
      /**
       * 存放测量信息全部overlay的索引数组，后面删除roi需要用
       * @type {Array}
       */
      measureOverlayIndexArray: {
        measureOverlayIndexArray : []
      },
      /**
       * 当前overlay的索引, 处理用户绘制需要用
       * @type {Number}
       */
      currentOverlayIndex: {
        currentOverlayIndex : -1
      },
      /**
       * 用户是否正在绘制ROI，处理用户绘制需要用
       * @type {Boolean}
       */
      isDrawingRoiFlag: {
        isDrawingRoiFlag : false
      },
       /**
       * 当前绘制的特征。
       * @type {import("../src/ol/Feature.js").default}
       */
      sketch: {
        sketch: null
      },
      /**
       * 帮助工具提示元素。
       * @type {HTMLElement}
       */
      helpTooltipElement: {
        helpTooltipElement: null
      },
      /**
       * 覆盖以显示帮助消息。
       * @type {Overlay}
       */
      helpTooltip: {
        helpTooltip: null
      },
      /**
       * 度量工具提示元素。
       * @type {HTMLElement}
       */
      measureTooltipElement: {
        measureTooltipElement : null
      },
      /**
       * 覆盖以显示测量值。
       * @type {Overlay}
       */
      measureTooltip: {
        measureTooltip : null
      },
       /**
       * 用户绘制多边形时显示的消息。
       * @type {string}
       */
      continuePolygonMsg: '单击以继续绘制多边形',
      pointerMoveHandler: null,
      source: {
        source : null
      },
      helpMsg : ""
    }
  },
  components: {
    //LayerSwitchControl: () => import("../components/LayerSwitchControl.vue"),
  },
  computed: {
    ...sync('app', [
      'navigationShowFlag','detailCardShowFlag', 'leftCardShowFlag',
      'arrowShowFlag','formShowFlag'
    ]),
    ...sync('form', [
      'formData','area','uploadFileShowFlag','coordinateArrayLength'
    ]),
   },
  // 初始化页面完成后，再对dom节点进行相关操作
  mounted(){
    this.source.source = new VectorSource();
    var this1 = this;   // 解决访问不到data中属性的问题(javascript基础薄弱)
    this.pointerMoveHandler = function (evt) {
    if (evt.dragging) {
      return;
    }
    /** @type {string} */
    this1.helpMsg = '单击以开始绘制';
    if (this1.sketch.sketch) {
      const geom = this1.sketch.sketch.getGeometry();
      if (geom instanceof Polygon) {
        this1.helpMsg = this1.continuePolygonMsg;
      }
    }
    this1.helpTooltipElement.helpTooltipElement.innerHTML = this1.helpMsg;
    this1.helpTooltip.helpTooltip.setPosition(evt.coordinate);
    this1.helpTooltipElement.helpTooltipElement.classList.remove('hidden');
    };
    
    //创建绘制按钮需要操作的变量通过字典传入，测试不通过这种方式变量值无法成功修改(javascript薄弱)
    let createDrawControlOptions = {
      "pointerMoveHandler" : this1.pointerMoveHandler,
      "source" : this1.source,
      "measureTooltipElement" : this1.measureTooltipElement,
      "measureTooltip" : this1.measureTooltip,
      "currentOverlayIndex" : this1.currentOverlayIndex,
      "measureOverlayIndexArray" : this1.measureOverlayIndexArray,
      "helpTooltipElement" : this1.helpTooltipElement,
      "helpTooltip" : this1.helpTooltip,
      "isDrawingRoiFlag" : this1.isDrawingRoiFlag,
      "sketch" : this1.sketch,
      "coordinateArray" : this1.coordinateArray,
      "userDrawnPolygons" : this1.userDrawnPolygons,
      "draw" : this1.draw,
      "roiArea" : this1.roiArea
    };
    //创建撤销绘制按钮需要操作的变量通过字典传入，测试不通过这种方式变量值无法成功修改(javascript基础薄弱)
    let backDrawControlOptions = {
      "draw" :  this1.draw,
    }
    //创建清空绘制按钮需要操作的变量通过字典传入，测试不通过这种方式变量值无法成功修改(javascript基础薄弱)
    let clearDrawControlOptions = {
      "draw" :  this1.draw,
      "isDrawingRoiFlag" : this1.isDrawingRoiFlag,
      "userDrawnPolygons" : this1.userDrawnPolygons,
      "source" : this1.source,
      "measureOverlayIndexArray" : this1.measureOverlayIndexArray,
      "currentOverlayIndex" : this1.currentOverlayIndex,
      "helpTooltipElement" : this1.helpTooltipElement,
    }
    
    let layersControlOptions = {
      "baseMap": this1.baseMap,
      "overlay": this1.overlay,
    }

    var format = 'image/png';
    this.layers = [
      //  OSM底图
      new TileLayer({
        source: new OSM(),
        visible: true,
        title: 'osmLayer',
        class: 'basemap' // 自定义属性：class（basemap/layer）  方便底图切换时,与要素图层区分开
      }),
      new ImageTile({
        title: 'waterextractcase',
        visible: false,
        source: new ImageWMS({
          ratio: 1,
          projectiton: 'EPSG:3857',
          url: 'https://geo.culturalmapofcdutyouth.com/geoserver/TestLoading/wms',
          params: {'FORMAT': format,
                'VERSION': '1.1.1',  
                "STYLES": '',
                "LAYERS": 'TestLoading:H48F019017_result',
                "exceptions": 'application/vnd.ogc.se_inimage',
          }
        },
        ),
        class: 'overlay',
      }),
      new VectorLayer({
        source: this.source.source,
        style: new Style({
          fill: new Fill({
            color: 'rgba(255, 255, 255, 0.2)',
          }),
          stroke: new Stroke({
            color: '#ffcc33',
            width: 2,
          }),
          image: new CircleStyle({
            radius: 7,
            fill: new Fill({
              color: '#ffcc33',
            }),
          }),
        }),
      }),
      //  google 栅格带标注 底图
      new TileLayer({
        title: 'hybridLayer_2020',
        visible: false,
        class: 'basemap',
        source: new XYZ({
          projectiton: 'EPSG:4326',
          crossOrigin: 'anonymous',
          url: 'http://mt2.google.cn/vt/lyrs=y&x={x}&y={y}&z={z}&s=G' //  标注地图 ，mt1/mt2/mt3 均可用
        })
      }),
      // google 栅格无标注 底图
      new TileLayer({
        title: 'satelliteLayer_2020',
        visible: false,
        class: 'basemap',
        source: new XYZ({
          projectiton: 'EPSG:4326',
          crossOrigin: 'anonymous',
          url: 'http://mt0.google.com/vt/lyrs=s&hl=en&x={x}&y={y}&z={z}' // 无标注地图
        })
      }),
      // esri-satellite 底图
      new TileLayer({
        title: 'esri-satellite',
        visible: false,
        class: 'basemap',
        source: new TileArcGISRest({
          crossOrigin: 'anonymous',
          projectiton: 'EPSG:4326',
          url: 'https://services.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer'
        })
      }),
      // stamen 底图
      new TileLayer({
        title: 'stamen',
        visible: false,
        class: 'basemap',
        source: new Stamen({
          layer: 'toner'
        })
      }),
      // new TileLayer({
      //   source: source,
      // })
    ]
    this.drawGeojson()

    this.map = new Map({
      // 添加绘制，撤销绘制，清空绘制控制
      controls: defaultControls({ attribution: false, zoom: true, rotate: false }).extend([new CreateDrawControl(createDrawControlOptions), new BackDrawControl(backDrawControlOptions), new ClearDrawControl(clearDrawControlOptions), new ScaleLine(), new LayersControl(layersControlOptions)]),
      layers: this.layers,
      target: 'map-container',
      view: new View({
        center: [11550688.058143763, 3612881.4592794343],
        zoom: 4,
      }),
    });
  
  /**
     * 手柄指针移动。
     * @param {import("../src/ol/MapBrowserEvent").default} evt The event.
     */
  let mouseoutthis = this;
  this.map.getViewport().addEventListener('mouseout', function () {
    mouseoutthis.helpTooltipElement.helpTooltipElement.classList.add('hidden');
  });
  this.drawGeojson();
},
methods: {
    // 根据测试的geojson在地图绘制面
    drawGeojson() {
      let features = [];
      for(let r = 0; r < this.geoJsonArray.length; r++){
        let polygoncoordinates = [];
        for(let i = 0; i < this.geoJsonArray[r]["geometry"]["coordinates"].length; i += 2) {
          if (i < this.geoJsonArray[r]["geometry"]["coordinates"].length - 1) {
            polygoncoordinates.push([this.geoJsonArray[r]["geometry"]["coordinates"][i], this.geoJsonArray[r]["geometry"]["coordinates"][i + 1]]);
          }
        }
        let featurePolygon = new Feature({
          geometry : new Polygon([polygoncoordinates]),
        })
        features.push(featurePolygon);
      }
      let layer = new VectorLayer();
      let source = new VectorSource();
      source.addFeatures(features);
      layer.setSource(source);
      layer.set('title', 'geoJson');
      layer.set('id', 'geoJson');
      layer.set('class', 'overlay');
      layer.set('visible', false);
      this.layers.push(layer);
    },

    setBasemap (title) {
      var layers = this.map.getLayers().array_
      // 循环遍历图层数组, 找到图层名为title的图层显示 --libin
      for (var i = 0; i < layers.length; i++) {
        // 图层类型为basemap才进行visible更改 --libin
        if (layers[i].values_.class === 'basemap') {
          if (layers[i].values_.title === title) {
              layers[i].setVisible(true);
          } else {
              layers[i].setVisible(false);
          }
        }
      }
    },

    setOverlay (title) {
      var layers = this.map.getLayers().array_;
      // 循环遍历图层数组, 找到图层名为title的图层显示 --libin
      for (var i = 0; i < layers.length; i++) {
        // 图层类型为basemap才进行visible更改 --libin
        if (layers[i].values_.class === 'overlay') {
          if (layers[i].values_.title === title) {
              layers[i].setVisible(true);
          } else {
              layers[i].setVisible(false);
          }
        }
      }
    },
    },

    watch: {
      // 同步更新用户绘制的roi的面积数据到formData
      roiArea: {
        deep: true,
        handler(nv) {
          this.formData.area = nv.roiArea;
        },
      },
      coordinateArray: {
        deep: true,
        handler(nv) {
          this.coordinateArrayLength = nv.length;
        },
      },
      baseMap: {
        deep: true,
        handler(nv) {
          this.setBasemap(nv.baseMap);
        },
      },
      overlay: {
        deep: true,
        handler(nv) {
          this.setOverlay(nv.overlay);
        },
      }
  },
}
</script>
<!-- 浏览代码可折叠 -->
<style scoped>
      .main {
        width: calc(100%);
        height: 100%
      }
      .map-container {
        position: absolute;
        left: 0;
        top: 0;
        width: calc(100%);
        height: 100%;
      }

      .bmapGem {
        border: 2px #ccc solid;
        margin: 3px;
        width: 100px;
        height: 60px;
        float: left;
        font-size: 12px;
        text-align: center;
        border-radius: 6px;
      }
      .overlayGem {
        border: 2px #ccc solid;
        margin: 3px;
        width: 100px;
        height: 60px;
        float: left;
        font-size: 12px;
        text-align: center;
        border-radius: 6px;
      }
      .bmImg {
        margin: 7px 7px 0px 7px;
      }
      .overlayImg{
        margin: 7px 7px 0px 7px;
      }
</style>
