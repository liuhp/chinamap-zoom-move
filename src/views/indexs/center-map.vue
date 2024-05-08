<template>
  <div class="centermap">
    <div style="width: 40%;">
      <div class="maptitle">
        <div class="zuo"></div>
        <span class="titletext">缩放轮播区域</span>
        <div class="you"></div>
      </div>
      <div style="display: flex;margin-bottom: 20px">
        <div style="margin-right: 8px;font-size: 20px;" >华北</div>
        <div style="margin-right: 8px;font-size: 20px;" >华东</div>
        <div style="margin-right: 8px;font-size: 20px;" >东北</div>
        <div style="margin-right: 8px;font-size: 20px;" >华中</div>
        <div style="margin-right: 8px;font-size: 20px;" >华南</div>
        <div style="margin-right: 8px;font-size: 20px;" >西南</div>
        <div style="margin-right: 8px;font-size: 20px;" >西北</div>
      </div>
      <div class="mapwrap">
        <dv-border-box-13>
          <div class="quanguo" @click="getData('china')" v-if="code !== 'china'">
            中国
          </div>

          <Echart id="CenterMap" :options="options" ref="CenterMap" />
        </dv-border-box-13>
      </div>
    </div>
    <div style="width: 40%;">
      <div class="maptitle">
        <div class="zuo"></div>
        <span class="titletext">切换展示区域</span>
        <div class="you"></div>
      </div>
      <div style="display: flex;margin-bottom: 20px">
        <div style="margin-right: 8px;cursor: pointer;font-size: 20px;" @click="changeArea('000001')">华北</div>
        <div style="margin-right: 8px;cursor: pointer;font-size: 20px;" @click="changeArea('000002')">华东</div>
        <div style="margin-right: 8px;cursor: pointer;font-size: 20px;" @click="changeArea('000003')">东北</div>
        <div style="margin-right: 8px;cursor: pointer;font-size: 20px;" @click="changeArea('000004')">华中</div>
        <div style="margin-right: 8px;cursor: pointer;font-size: 20px;" @click="changeArea('000005')">华南</div>
        <div style="margin-right: 8px;cursor: pointer;font-size: 20px;" @click="changeArea('000006')">西南</div>
        <div style="margin-right: 8px;cursor: pointer;font-size: 20px;" @click="changeArea('000007')">西北</div>
      </div>
      <div class="mapwrap">
        <dv-border-box-13>
          <div class="quanguo" @click="getData('china')" v-if="code !== 'china'">
            中国
          </div>

          <Echart id="CenterMap2" :options="options2" ref="CenterMap2" />
        </dv-border-box-13>
      </div>
    </div>
    
  </div>
</template>

<script>
import xzqCode from "../../utils/map/xzqCode";
import { currentGET } from "api/modules";
import * as echarts from "echarts";
import { GETNOBASE } from "api";
import { deepClone } from '@/utils/index'
export default {
  data() {
    return {
      maptitle: "中国地图",
      options: {},
      options2:{},
      code: "china", //china 代表中国 其他地市是行政编码
      echartBindClick: false,
      isSouthChinaSea: false, //是否要展示南海群岛  修改此值请刷新页面
      regions: [
        {name: '中国', center: [104.195397, 35.86166], zoom: 1},
        {name: '华北', center: [116.397428, 39.90923], zoom: 2},
        {name: '华东', center: [121.473701, 31.230416], zoom: 2},
        {name: '东北', center: [123.431475, 41.805698], zoom: 2},
        {name: '华中', center: [114.305392, 30.593099], zoom: 2},
        {name: '华南', center: [113.280637, 23.125178], zoom: 2},
        {name: '西南', center: [104.065735, 30.659462], zoom: 2},
        {name: '西北', center: [95.948021, 34.263161], zoom: 2},
        
      ],
      zoom:1,
      center:[104.195397, 35.86166],
      intervalTimer: null,
    };
  },
  created() {},
  beforeDestroy(){
    this.intervalTimer = null
  },
  mounted() {
    // console.log(xzqCode);
    this.getData("china");
  },
  methods: {
    getData(code, isStop) {
      currentGET("big8", { regionCode: code }).then((res) => {
        if (res.success) {
          this.getGeojson(res.data.regionCode, res.data.dataList);
          this.mapclick();
        } else {
          this.$Message.warning(res.msg);
        }
      });
    },
    /**
     * @description: 获取geojson
     * @param {*} name china 表示中国 其他省份行政区编码
     * @param {*} mydata 接口返回列表数据
     * @return {*}
     */
    async getGeojson(name, mydata) {
      this.code = name;
      //如果要展示南海群岛并且展示的是中国的话
      let geoname=name
      if (this.isSouthChinaSea && name == "china") {
        geoname = "chinaNanhai";
      }
      //如果有注册地图的话就不用再注册 了
      let mapjson = echarts.getMap(name);
      if (mapjson) {
        mapjson = mapjson.geoJSON;
      } else {
        mapjson = await GETNOBASE(`./map-geojson/${geoname}.json`).then((res) => {
          return res;
        });
        echarts.registerMap(name, mapjson);
      }
      let cityCenter = {};
      let arr = mapjson.features;
      //根据geojson获取省份中心点
      arr.map((item) => {
        cityCenter[item.properties.name] =
          item.properties.centroid || item.properties.center;
      });
      let newData = [];
      mydata.map((item) => {
        if (cityCenter[item.name]) {
          newData.push({
            name: item.name,
            value: cityCenter[item.name].concat(item.value),
          });
        }
      });
      this.init(name, mydata, newData);
      this.interVal()
    },
    // 轮播缩放
    interVal(){
      let currentIndex = 0;
      this.intervalTimer = setInterval(() => {
          let currentRegion = this.regions[currentIndex];

          this.options.series[0].zoom = currentRegion.zoom
          this.options.series[0].center = currentRegion.center
          this.options.series[1].zoom = currentRegion.zoom
          this.options.series[1].center = currentRegion.center

          currentIndex = (currentIndex + 1) % this.regions.length;

      }, 3000);
    },
    init(name, data, data2) {
      let top = 45;
      let zoom = 1.05;
      let option = {
        backgroundColor: "rgba(0,0,0,0)",
        tooltip: {
          show: false,
        },
        legend: {
          show: false,
        },
        visualMap: {
          left: 20,
          bottom: 20,
          pieces: [
            { gte: 1000, label: "1000个以上" }, // 不指定 max，表示 max 为无限大（Infinity）。
            { gte: 600, lte: 999, label: "600-999个" },
            { gte: 200, lte: 599, label: "200-599个" },
            { gte: 50, lte: 199, label: "49-199个" },
            { gte: 10, lte: 49, label: "10-49个" },
            { lte: 9, label: "1-9个" }, // 不指定 min，表示 min 为无限大（-Infinity）。
          ],
          inRange: {
            // 渐变颜色，从小到大
            color: [
              "#c3d7df",
              "#5cb3cc",
              "#8abcd1",
              "#66a9c9",
              "#2f90b9",
              "#1781b5",
            ],
          },
          textStyle: {
            color: "#fff",
          },
        },
        geo: {
          map: name,
          roam: false,
          selectedMode: false, //是否允许选中多个区域
          zoom: zoom,
          top: top,
          // aspectScale: 0.78,
          show: false,
        },
        series: [
          {
            name: "MAP",
            type: "map",
            map: name,
            // aspectScale: 0.78,
            data: data,
            // data: [1,100],
            selectedMode: false, //是否允许选中多个区域
            zoom: this.zoom,
            center: this.center,
            geoIndex: 2,
            top: top,
            tooltip: {
              show: true,
              formatter: function (params) {
                if (params.data) {
                  return params.name + "：" + params.data["value"];
                } else {
                  return params.name;
                }
              },
              backgroundColor: "rgba(0,0,0,.6)",
              borderColor: "rgba(147, 235, 248, .8)",
              textStyle: {
                color: "#FFF",
              },
            },
            label: {
              show: true,
              color: "#FFF",
              // position: [-10, 0],
              formatter: function (val) {
                // console.log(val)
                if (val.data !== undefined) {
                  return val.name.slice(0, 2);
                } else {
                  return "";
                }
              },
              rich: {},
            },
            emphasis: {
              disabled: false,
              label: {
                show: true,
              },
              itemStyle: {
                areaColor: "#389BB7",
                borderWidth: 1,
              },
            },
            itemStyle: {
              color: '#fff',
              borderColor: "rgba(147, 235, 248, .8)",
              borderWidth: 1,
              areaColor: {
                type: "radial",
                x: 0.5,
                y: 0.5,
                r: 0.8,
                colorStops: [
                  {
                    offset: 0,
                    color: "rgba(147, 235, 248, 0)", // 0% 处的颜色
                  },
                  {
                    offset: 1,
                    color: "rgba(147, 235, 248, .2)", // 100% 处的颜色
                  },
                ],
                globalCoord: false, // 缺为 false
              },
              shadowColor: "rgba(128, 217, 248, .3)",
              shadowOffsetX: -2,
              shadowOffsetY: 2,
              shadowBlur: 10,
            },
          },
          {
            data: data2,
            type: "effectScatter",
            coordinateSystem: "geo",
            symbolKeepAspect: true,
            symbolSize: function (val) {
              return 0;
              // return val[2] / 50;
            },
            legendHoverLink: true,
            showEffectOn: "render",
            rippleEffect: {
              // period: 4,
              scale: 6,
              color: "rgba(255,255,255, 1)",
              brushType: "fill",
            },
            tooltip: {
              show: true,
              formatter: function (params) {
                if (params.data) {
                  return params.name + "：" + params.data["value"][2];
                } else {
                  return params.name;
                }
              },
              backgroundColor: "rgba(0,0,0,.6)",
              borderColor: "rgba(147, 235, 248, .8)",
              textStyle: {
                color: "#FFF",
              },
            },
            label: {
              formatter: (param) => {
                return param.name.slice(0, 2);
              },

              fontSize: 11,
              offset: [0, 2],
              position: "bottom",
              textBorderColor: "#fff",
              textShadowColor: "#000",
              textShadowBlur: 10,
              textBorderWidth: 0,
              color: "#FFF",
              show: false,
            },
            // colorBy: "data",
            itemStyle: {
              color: "rgba(255,255,255,1)",
              borderColor: "rgba(2255,255,255,2)",
              borderWidth: 4,
              shadowColor: "#000",
              shadowBlur: 10,
            },
          },
        ],
        //  动画效果
            // animationDuration: 1000,
            animationEasing: 'linear',
            animationDurationUpdate: 1000
      };
      this.options = deepClone(option);
      this.options2 = deepClone(option);
    },
    message(text) {
      this.$Message({
        text: text,
        type: "warning",
      });
    },
    changeArea(adcode){
      this.getData(adcode, false);
    },
    mapclick() {
      if (this.echartBindClick) return;
      //单击切换到级地图，当mapCode有值,说明可以切换到下级地图
      this.$refs.CenterMap.chart.on("click", (params) => {
        // console.log(params);
        let xzqData = xzqCode[params.name];
        if (xzqData) {
          this.getData(xzqData.adcode);
        } else {
          this.message("暂无下级地市!");
        }
      });
      this.echartBindClick = true;
    },
  },
};
</script>
<style lang="scss" scoped>
.centermap {
  margin-bottom: 30px;
  display: flex;
  justify-content: space-around;

  .maptitle {
    height: 60px;
    display: flex;
    justify-content: center;
    padding-top: 10px;
    box-sizing: border-box;

    .titletext {
      font-size: 28px;
      font-weight: 900;
      letter-spacing: 6px;
      background: linear-gradient(
        92deg,
        #0072ff 0%,
        #00eaff 48.8525390625%,
        #01aaff 100%
      );
      -webkit-background-clip: text;
      -webkit-text-fill-color: transparent;
      margin: 0 10px;
    }

    .zuo,
    .you {
      background-size: 100% 100%;
      width: 29px;
      height: 20px;
      margin-top: 8px;
    }

    .zuo {
      background: url("../../assets/img/xiezuo.png") no-repeat;
    }

    .you {
      background: url("../../assets/img/xieyou.png") no-repeat;
    }
  }

  .mapwrap {
    height: 548px;
    width: 100%;
    // padding: 0 0 10px 0;
    box-sizing: border-box;
    position: relative;

    .quanguo {
      position: absolute;
      right: 20px;
      top: -46px;
      width: 80px;
      height: 28px;
      border: 1px solid #00eded;
      border-radius: 10px;
      color: #00f7f6;
      text-align: center;
      line-height: 26px;
      letter-spacing: 6px;
      cursor: pointer;
      box-shadow: 0 2px 4px rgba(0, 237, 237, 0.5),
        0 0 6px rgba(0, 237, 237, 0.4);
    }
  }
}
</style>
