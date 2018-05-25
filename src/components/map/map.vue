<template>
  <div class="map-wrapper">
    <div id="map">
    </div>
    <Slider :value.sync="range" class="slide" @on-change="generlizeChange"></Slider>
    <Modal v-model="modal" title="配置数据选项" width="600" v-if="showConfigDialog">
      <Form :model="formItem" :label-width="80">
        <FormItem :label="index" v-for="(ftype, index) in featureTypes" :key="index">
          <v-switch @click.native="change(index)">
              <span slot="open">All</span>
          </v-switch>
          <CheckboxGroup v-model="formItem[index]">
            <Checkbox :label="item" v-for="item in ftype" :key="item"></Checkbox>
          </CheckboxGroup>
        </FormItem>
      </Form>
      <div class="data">
        {{formItem}}
      </div>
    </Modal>
  </div>
</template>

<script>
import {featureType} from '../../common/config/config'
import JQuery, {ajax} from 'jquery'
import L from 'leaflet'
import osmtogeojson from 'osmtogeojson'
import 'leaflet-draw'
import {Modal, Button, Form, FormItem, Checkbox, CheckboxGroup, Switch, Slider} from 'iview'
const simplify = require('simplify-geojson')
let map = null
let drawnItems = null
let OSMLayer = null
let googleLayer = null
let polygonObj = {}
let fetchData = null
let simplifiedLayer = null
let simplifiedData = null
export default {
  components: {
    Modal,
    Button,
    Form,
    FormItem,
    Checkbox,
    CheckboxGroup,
    vSwitch: Switch,
    Slider
  },
  data () {
    return {
      modal: true,
      switch1: false,
      formItem: {},
      formitems: [],
      featureTypes: {},
      showConfigDialog: false,
      range: 25
    }
  },
  mounted () {
    this._mapInit()
    this._drawRectForData()
    this.eClickRect()
  },
  created () {
    this.featureTypes = featureType
    Object.keys(featureType).forEach((item, index) => {
      this.$set(this.formItem, item, [])
      this.$set(this.formItem[item], 'flag', false)
    })
  },
  methods: {
    generlizeChange (value) {
      let formateRange = value / 100 * (0.001 - 0.0001) + 0.0001
      if (simplifiedLayer) {
        map.removeLayer(simplifiedLayer)
        simplifiedData = simplify(fetchData, formateRange)
        simplifiedLayer = L.geoJSON(simplifiedData, {
          style: function (feature) {
            return {color: 'red'}
          }
        })
        map.addLayer(simplifiedLayer)
      }
    },
    change (index) {
      this.formItem[index].flag = !this.formItem[index].flag
      if (this.formItem[index].flag) {
        this.formItem[index] = this.featureTypes[index]
      } else {
        this.formItem[index] = []
      }
    },
    // 点击画矩形框时出现edit按钮
    eClickRect () {
      let elRect = document.getElementsByClassName('leaflet-draw-draw-rectangle')[0]
      elRect.addEventListener('click', (e) => {
        let tarEl = document.getElementsByClassName('leaflet-draw-section')[1]
        this.showConfigDialog = true
        tarEl.style.display = 'block'
      })
    },
    // ajax请求数据
    getData (data, polygonObj) {
      let that = this
      ajax({
        type: 'POST',
        url: 'http://overpass-api.de/api/interpreter',
        data: {
          data: data
        },
        dataType: 'json',
        success: function (resdata) {
          resdata = osmtogeojson(resdata)
          fetchData = resdata
          that.add({
            title: 'download origin data',
            text: 'DR',
            classname: 'btn download',
            download: 'raw_data.geojson',
            href: URL.createObjectURL(new Blob([JSON.stringify(fetchData)]))
          })
          let jsonLayer = L.geoJSON(resdata)
          polygonObj.layer = jsonLayer
          jsonLayer.addTo(map)
        }
      })
    },
    // 地图初始化部分
    _mapInit () {
      let OSMOptions = {
        url: 'http://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png',
        attr: '&copy; <a href="http://openstreetmap.org/copyright">OpenStreetMap</a> contributors'
      }
      OSMLayer = L.tileLayer(OSMOptions.url, { maxZoom: 18, attribution: OSMOptions.attr })
      googleLayer = L.tileLayer('http://www.google.cn/maps/vt?lyrs=s@189&gl=cn&x={x}&y={y}&z={z}', {
        attribution: 'google'
      })
      map = new L.Map('map', { center: new L.LatLng(51.505, -0.04), zoom: 13 })
      // this._GeneralizeControl(map)
      drawnItems = L.featureGroup().addTo(map)
      L.control.layers({
        'osm': OSMLayer.addTo(map),
        'google': googleLayer.addTo(map)
      }, { 'drawlayer': drawnItems }, { position: 'topleft', collapsed: true }).addTo(map)
      map.addControl(new L.Control.Draw({
        edit: {
          featureGroup: drawnItems,
          poly: {
            allowIntersection: false
          }
        },
        draw: {
          polygon: false,
          marker: false,
          circle: false,
          polyline: false,
          circlemarker: false
        }
      }))
      this._noLayer()
      // this.addBtn()
      this.add({
        callback: (el) => {
          let slideTar = document.getElementsByClassName('slide')[0]
          el.addEventListener('click', () => {
            slideTar.style.display = 'block'
            simplifiedData = simplify(fetchData, 0.0002)
            simplifiedLayer = L.geoJSON(simplifiedData, {
              style: function (feature) {
                return {color: 'red'}
              }
            })
            map.addLayer(simplifiedLayer)
            this.add({
              title: 'download origin data',
              classname: 'btn download',
              text: 'DS',
              download: 'simplified_data.geojson',
              href: URL.createObjectURL(new Blob([JSON.stringify(simplifiedData)]))
            })
          }, false)
        },
        title: 'simplify',
        href: '#',
        text: 'S',
        classname: 'btn simplify'
      })
    },
    // 画矩形框时需要的事件绑定操作
    _drawRectForData () {
      map.on(L.Draw.Event.CREATED, (event) => {
        // 这里的layer是画的矩形，需要根据这个矩形的bound来获取数据
        let layer = event.layer
        // 获取画出的矩形的id
        let layerId = L.Util.stamp(layer)
        // 记录画出的每个矩形，便于在edit后获取所画的矩形对象
        polygonObj[layerId] = {bounds: event.layer._bounds}
        // 让画出的矩形显示出来
        drawnItems.addLayer(layer)
        // 获取数据及显示
        let dataStr = this._getDataStr(layer)
        this.getData(dataStr, polygonObj[layerId])
      })
      map.on(L.Draw.Event.EDITED, (event) => {
        // 由于这里的event中的layers获取到的是map中所有的图层，因此需要遍历判断一下，这也是设计polygonObj的原因
        for (let layerId in polygonObj) {
          // 获取edit后的画出的矩形
          let layer = event.layers._layers[layerId]
          if (layer) {
            // edit后矩形的bound
            let bounds = JSON.stringify(layer._bounds)
            // 如果二者不一致，说明该矩形被edit过
            if (!(JSON.stringify(polygonObj[layerId].bounds) === bounds)) {
              // 移除该由该矩形获取的数据，并且map不再显示该数据
              map.removeLayer(polygonObj[layerId].layer)
              // 重新获取数据并显示
              let dataStr = this._getDataStr(layer)
              this.getData(dataStr, polygonObj[L.Util.stamp(layer)])
            }
          }
        }
      })
    },
    // 根据画出的矩形范围获取请求数据的拼接字符串
    _getDataStr (layer) {
      let coordinate = [layer._bounds._southWest.lat, layer._bounds._southWest.lng, layer._bounds._northEast.lat, layer._bounds._northEast.lng]
      let rangeStr = ''
      for (let index in this.formItem) {
        if (this.formItem[index].length > 0) {
          rangeStr += 'node["' + index.toLowerCase() + '"](' + coordinate + ');'
          rangeStr += 'way["' + index.toLowerCase() + '"](' + coordinate + ');'
          rangeStr += 'relation["' + index.toLowerCase() + '"](' + coordinate + ');'
        }
      }
      let data = '[out:json];(' + rangeStr + ');out;>;out skel qt;'
      return data
    },
    // 不需要显示底图
    _noLayer () {
      let domObj = document.getElementsByClassName('leaflet-control-layers-base')[0]
      let addBtn = document.createElement('label')
      addBtn.innerHTML = '<div><input type="radio" name="leaflet-base-layers" class="leaflet-control-layers-selector no-layer-hook"><span> no-layer </span></div>'
      domObj.appendChild(addBtn)
      document.getElementsByClassName('no-layer-hook')[0].addEventListener('change', (e) => {
        map.removeLayer(googleLayer)
        map.removeLayer(OSMLayer)
        domObj.appendChild(addBtn)
      }, false)
    },
    add (options = {callback: null, title: '', href: '#', classname: '', text: '', download: ''}) {
      var container = JQuery('.leaflet-draw-edit-edit').parent()[0]
      let el = document.createElement('a')
      el.title = options.title
      el.href = options.href
      el.className = options.classname
      el.download = options.download
      el.innerHTML = '<span class="color">' + options.text + '</span>'
      container.appendChild(el)
      if (options.callback) {
        options.callback(el)
      }
    }
    // addBtn (options) {
    //   var container = JQuery('.leaflet-draw-edit-edit').parent()[0]
    //   let downLoadOriginEl = document.createElement('a')
    //   downLoadOriginEl.title = 'download origin data'
    //   downLoadOriginEl.className = 'btn download'
    //   downLoadOriginEl.download = 'raw_data.geojson'
    //   downLoadOriginEl.href = URL.createObjectURL(new Blob([fetchData]))
    //   downLoadOriginEl.innerHTML = '<span class="color">DR</span>'
    //   container.appendChild(downLoadOriginEl)

    //   let downLoadSimEl = document.createElement('a')
    //   downLoadSimEl.title = 'download origin data'
    //   downLoadSimEl.className = 'btn download'
    //   downLoadSimEl.innerHTML = '<span class="color">DS</span>'
    //   container.appendChild(downLoadSimEl)
    // }
  }
}
</script>

<style lang="less">
@import 'leaflet/dist/leaflet.css';
@import 'leaflet-draw/dist/leaflet.draw.css';
.clearfix{
  content:".";
  display:block;
  height:0;
  clear:both;
  visibility:hidden
}
.slide{
  display: none;
  position: absolute;
  top: 190px;
  left: 12px;
  z-index: 20000;
  width: 180px;
}
.btn{
  float: left;
  width: 30px;
  height: 30px;
  background: 2px solid rgba(0,0,0,0.2);
  border-radius: 2px;
  margin-left: 2px;
  text-align: center;
  line-height: 31px;
  font-size: 20px;
  background: #fff!important;
  border: 1px solid #ccc!important;
  margin-left: -1px;
  .color{
    color: rgba(0,0,0,0.8)!important;
  }
}
.download{
  font-size: 14px;
}
.map-wrapper{
  height: 100%;
  position: relative;
  #map{
    height: 100%;
    width: 100%;
  }
  .leaflet-control-container{
    &::after{
      .clearfix();
    }
  }
  .leaflet-touch .leaflet-control-layers-toggle {
      width: 30px;
      height: 30px;
  }
  .leaflet-control-layers-expanded{
    .leaflet-control-layers-overlays{
      display: none;
    }
    .leaflet-control-layers-separator{
      display: none;
    }
  }
  .leaflet-draw-toolbar-top{
    width: 34px;
  }
  .leaflet-draw-section{
    &:nth-child(2){
      float: left!important;
      display: none;
    }
    .leaflet-draw-toolbar{
      margin-top: 0;
      overflow: hidden;
      .leaflet-draw-edit-edit{
        float: left;
      }
      .leaflet-draw-edit-remove{
        float: left;
      }
    }
    .leaflet-bar a, .leaflet-bar a:hover{
      border-bottom: none;
      border-right: 1px solid #ccc;
    }
    .leaflet-bar{
      a:nth-child(last){
        border-bottom: none;
        border-right: none;
        &:hover{
          border-bottom: none;
          border-right: none;
        }
      }
    }
  }
  .leaflet-bar a.leaflet-disabled {
    cursor: pointer;
  }
  .leaflet-draw-actions-top.leaflet-draw-actions-bottom a {
    height: 30px;
    line-height: 30px;
  }
}
.ivu-form .ivu-form-item-label{
  text-align: left;
  font-weight: 700;
}
</style>
