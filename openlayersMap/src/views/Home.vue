<template>
  <div class="home">
    <div class="btns">
      <el-button @click="addPointMode">添加点</el-button>
      <el-button @click="deletePoint">删除点</el-button>
      <el-button @click="toPoint">定位到某个点</el-button>
      <el-button @click="planRoute">路径规划</el-button>
      <el-button @click="addText">添加文字标注</el-button>
    </div>
    <div class="map" id="map"></div>
    <!-- 点信息显示框 - 移到底部 -->
    <div class="point-info-container bottom-position">
      <h3>点信息</h3>
      <el-table
        ref="pointTable"
        :data="points"
        border
        @selection-change="handleSelectionChange"
        style="width: 100%">
        <el-table-column type="selection" width="55"></el-table-column>
        <el-table-column prop="id" label="ID" width="80"></el-table-column>
        <el-table-column prop="longitude" label="经度"></el-table-column>
        <el-table-column prop="latitude" label="纬度"></el-table-column>
      </el-table>
    </div>
  </div>
</template>

<script>
import * as ol from 'openlayers';

export default {
  data() {
    return {
      map: null,
      popupContentShow: false,
      layerList: [],
      // 点信息管理
      points: [],
      nextPointId: 1,
      selectedPoints: [],
      isAddingPoints: false,
      pointVectorSource: null,
      pointVectorLayer: null,
      // 新增：用于高亮显示的图层
      highlightLayer: null,
      highlightSource: null,
      // 新增：路径规划相关
      routeSource: null,
      routeLayer: null,
      // 优化：添加加载状态
      isPlanningRoute: false,
    };
  },
  mounted() {
    // 生成地图
    this.initMap();
    // 初始化点图层
    this.initPointLayer();
    // 初始化高亮图层
    this.initHighlightLayer();
    // 初始化路径图层
    this.initRouteLayer();
  },
  methods: {
    clearMap() {
      if (this.layerList.length) {
        this.layerList.forEach((item) => {
          this.map.removeLayer(item);
        });
      }
      this.map.getOverlays().clear();
    },
    initMap() {
      var projection = ol.proj.get('EPSG:3857');
      //分辨率
      var resolutions = [];
      for (var i = 0; i < 19; i++) {
        resolutions[i] = Math.pow(2, 18 - i);
      }
      var tilegrid = new ol.tilegrid.TileGrid({
        origin: [0, 0],
        resolutions: resolutions,
      });
      //拼接百度地图出图地址
      var baidu_source = new ol.source.TileImage({
        //设置坐标参考系
        projection: projection,
        //设置分辨率
        tileGrid: tilegrid,
        //出图基地址
        tileUrlFunction: function (tileCoord, pixelRatio, proj) {
          if (!tileCoord) {
            return '';
          }
          var z = tileCoord[0];
          var x = tileCoord[1];
          var y = tileCoord[2];

          if (x < 0) {
            x = 'M' + -x;
          }
          if (y < 0) {
            y = 'M' + -y;
          }
          return (
            'http://online3.map.bdimg.com/onlinelabel/?qt=tile&x=' +
            x +
            '&y=' +
            y +
            '&z=' +
            z +
            '&styles=pl&udt=20151021&scaler=1&p=1'
          );
        },
      });
      //百度地图
      var baidu_layer = new ol.layer.Tile({
        source: baidu_source,
      });
      //地图容器
      this.map = new ol.Map({
        target: 'map',
        layers: [baidu_layer],
        view: new ol.View({
          center: ol.proj.transform([116.403, 39.924], 'EPSG:4326', 'EPSG:3857'),
          zoom: 12,
          minZoom: 3,
        }),
      });
    },
    moveToPosition(postion, zoom = 14) {
      this.map.getView().animate({
        //将地理坐标转为投影坐标
        center: ol.proj.transform(postion, 'EPSG:4326', 'EPSG:3857'),
        duration: 500,
        zoom: zoom,
      });
    },
    addPoint() {
      this.clearMap();
      //创建一个点
      var point = new ol.Feature({
        geometry: new ol.geom.Point(ol.proj.transform([116.403, 39.924], 'EPSG:4326', 'EPSG:3857')),
      });
      //设置点1的样式信息
      point.setStyle(
        new ol.style.Style({
          //填充色
          fill: new ol.style.Fill({
            color: 'rgba(255, 255, 255, 0.2)',
          }),
          //边线颜色
          stroke: new ol.style.Stroke({
            color: '#ffcc33',
            width: 2,
          }),
          //形状
          image: new ol.style.Circle({
            radius: 17,
            fill: new ol.style.Fill({
              color: '#ffcc33',
            }),
          }),
        })
      );

      //实例化一个矢量图层Vector作为绘制层
      var source = new ol.source.Vector({
        features: [point],
      });
      console.log(source);
      //创建一个图层
      var vector = new ol.layer.Vector({
        source: source,
      });
      //将绘制层添加到地图容器中
      this.map.addLayer(vector);

      // 移动位置
      this.moveToPosition([116.403, 39.924]);

      // 将已添加的图层装起来
      this.layerList.push(vector);
    },
    toPoint() {
      // 如果有选中的点
      if (this.selectedPoints.length > 0) {
        // 找出ID最小的点
        let minIdPoint = this.selectedPoints.reduce((prev, current) => {
          return (prev.id < current.id) ? prev : current;
        });
        
        // 移动到该点
        this.moveToPosition([parseFloat(minIdPoint.longitude), parseFloat(minIdPoint.latitude)]);
        
        // 高亮显示该点
        this.highlightPoint(minIdPoint.point);
      } else {
        // 没有选中的点，使用默认行为
        this.map.getView().animate({
          center: ol.proj.transform([117.403, 42.924], 'EPSG:4326', 'EPSG:3857'),
          duration: 1000,
          zoom: 12,
        });
      }
    },
    // 新增：初始化高亮图层
    initHighlightLayer() {
      this.highlightSource = new ol.source.Vector();
      this.highlightLayer = new ol.layer.Vector({
        source: this.highlightSource,
        style: new ol.style.Style({
          stroke: new ol.style.Stroke({
            color: 'red',
            width: 3,
          }),
          image: new ol.style.Circle({
            radius: 20,
            fill: new ol.style.Fill({
              color: 'rgba(255, 0, 0, 0.2)',
            }),
            stroke: new ol.style.Stroke({
              color: 'red',
              width: 2,
            }),
          }),
        }),
      });
      this.map.addLayer(this.highlightLayer);
    },
    // 新增：高亮显示指定点
    highlightPoint(coordinate) {
      // 清空之前的高亮
      this.highlightSource.clear();
      
      // 创建高亮要素
      var highlightFeature = new ol.Feature({
        geometry: new ol.geom.Point(coordinate),
      });
      
      // 添加到高亮图层
      this.highlightSource.addFeature(highlightFeature);
    },
    addText() {
      // 清除绘制的东西
      this.clearMap();
      /**
       * 创建矢量标注样式函数,设置image为图标ol.style.Icon
       * @param {ol.Feature} feature 要素
       */
      var createLabelStyle = function (feature) {
        return new ol.style.Style({
          text: new ol.style.Text({
            //位置
            textAlign: 'center',
            //基准线
            textBaseline: 'middle',
            //文字样式
            font: 'normal 14px 微软雅黑',
            //文本内容
            text: feature.get('name'),
            //文本填充样式（即文字颜色）
            fill: new ol.style.Fill({ color: '#aa3300' }),
            stroke: new ol.style.Stroke({ color: '#ffcc33', width: 2 }),
          }),
        });
      };

      //实例化Vector要素，通过矢量图层添加到地图容器中
      var iconFeature = new ol.Feature({
        geometry: new ol.geom.Point(ol.proj.transform([116.403, 39.924], 'EPSG:4326', 'EPSG:3857')),
        //名称属性
        name: '北京市',
        //大概人口数（万）
        population: 2115,
      });
      iconFeature.setStyle(createLabelStyle(iconFeature));
      //矢量标注的数据源
      var vectorSource = new ol.source.Vector({
        features: [iconFeature],
      });
      //矢量标注图层
      var vectorLayer = new ol.layer.Vector({
        source: vectorSource,
      });
      this.map.addLayer(vectorLayer);
      this.moveToPosition([116.403, 39.924]);
      // 将已添加的图层装起来
      this.layerList.push(vectorLayer);
    },
    createLabelStyle(feature) {
      return new ol.style.Style({
        /**{olx.style.IconOptions}类型*/
        image: new ol.style.Icon({
          anchor: [0.5, 60],
          anchorOrigin: 'top-right',
          anchorXUnits: 'fraction',
          anchorYUnits: 'pixels',
          offsetOrigin: 'top-right',
          // offset:[0,10],
          //图标缩放比例
          // scale:0.5,
          //透明度
          opacity: 0.95,
          //图标的url
          src: "../assets/logo.png",
        }),
      });
    },
    addVectorLabel(coordinate, vectorSource) {
      // 新建一个要素 ol.Feature
      var newFeature = new ol.Feature({
        //几何信息
        geometry: new ol.geom.Point(coordinate),
      });

      // 设置点的样式信息
      newFeature.setStyle(
        new ol.style.Style({
          //填充色
          fill: new ol.style.Fill({
            color: 'rgba(255, 255, 255, 0.2)',
          }),
          //边线颜色
          stroke: new ol.style.Stroke({
            color: '#ffcc33',
            width: 2,
          }),
          //形状
          image: new ol.style.Circle({
            radius: 17,
            fill: new ol.style.Fill({
              color: '#ffcc33',
            }),
          }),
        })
      );

      // 实例化一个矢量图层Vector作为绘制层
      if (!vectorSource) {
        vectorSource = new ol.source.Vector();
        var vector = new ol.layer.Vector({
          source: vectorSource,
        });
        this.map.addLayer(vector);
        this.layerList.push(vector);
      }
      vectorSource.addFeature(newFeature);
    },
    // 初始化点图层
    initPointLayer() {
      this.pointVectorSource = new ol.source.Vector();
      this.pointVectorLayer = new ol.layer.Vector({
        source: this.pointVectorSource,
        style: new ol.style.Style({
          fill: new ol.style.Fill({
            color: 'rgba(255, 255, 255, 0.2)',
          }),
          stroke: new ol.style.Stroke({
            color: '#ffcc33',
            width: 2,
          }),
          image: new ol.style.Circle({
            radius: 17,
            fill: new ol.style.Fill({
              color: '#ffcc33',
            }),
          }),
        }),
      });
      this.map.addLayer(this.pointVectorLayer);
      this.layerList.push(this.pointVectorLayer);
    },
    // 切换添加点模式
    addPointMode() {
      this.isAddingPoints = !this.isAddingPoints;
      if (this.isAddingPoints) {
        // 注册地图点击事件
        this.map.on('click', this.handleMapClick);
        alert('已进入添加点模式，请在地图上点击添加点');
      } else {
        // 移除地图点击事件
        this.map.un('click', this.handleMapClick);
        alert('已退出添加点模式');
      }
    },
    // 处理地图点击事件
    handleMapClick(evt) {
      const point = evt.coordinate;
      // 将地理坐标从投影坐标转换为EPSG:4326
      const coord4326 = ol.proj.transform(point, 'EPSG:3857', 'EPSG:4326');
      
      // 创建点对象并添加到列表
      const newPoint = {
        id: this.nextPointId++,
        longitude: coord4326[0].toFixed(6),
        latitude: coord4326[1].toFixed(6),
        point: point
      };
      this.points.push(newPoint);
      
      // 在地图上添加点
      this.addVectorLabel(point, this.pointVectorSource);
    },
    // 处理表格选择变化
    handleSelectionChange(selection) {
      this.selectedPoints = selection;
    },
    // 删除选中的点
    deletePoint() {
      if (this.selectedPoints.length === 0) {
        alert('请先选择要删除的点');
        return;
      }
      
      // 确认删除
      if (confirm(`确定要删除选中的 ${this.selectedPoints.length} 个点吗？`)) {
        // 删除地图上的点
        this.selectedPoints.forEach(point => {
          // 找到对应的feature并删除
          const features = this.pointVectorSource.getFeatures();
          features.forEach(feature => {
            const coord = feature.getGeometry().getCoordinates();
            if (coord[0] === point.point[0] && coord[1] === point.point[1]) {
              this.pointVectorSource.removeFeature(feature);
            }
          });
          
          // 从表格数据中删除
          const index = this.points.findIndex(p => p.id === point.id);
          if (index !== -1) {
            this.points.splice(index, 1);
          }
        });
        
        // 清空选中
        this.$refs.pointTable.clearSelection();
        alert('删除成功');
      }
    },
    // 新增：初始化路径图层
    initRouteLayer() {
      this.routeSource = new ol.source.Vector();
      this.routeLayer = new ol.layer.Vector({
        source: this.routeSource,
        style: new ol.style.Style({
          stroke: new ol.style.Stroke({
            color: 'red',
            width: 3,
          }),
        }),
      });
      this.map.addLayer(this.routeLayer);
      this.layerList.push(this.routeLayer);
    },
    // 优化：路径规划方法
    planRoute() {
      // 检查是否已经在规划路径
      if (this.isPlanningRoute) {
        alert('正在规划路径，请稍候...');
        return;
      }
      
      // 设置加载状态
      this.isPlanningRoute = true;
      
      try {
        // 清空之前的路径
        this.routeSource.clear();
        
        // 检查是否选择了足够的点
        if (this.selectedPoints.length < 2) {
          alert('请至少选择2个点进行路径规划');
          this.isPlanningRoute = false;
          return;
        }
        
        // 按照ID从小到大排序
        const sortedPoints = [...this.selectedPoints].sort((a, b) => a.id - b.id);
        
        // 提取坐标点
        const coordinates = sortedPoints.map(point => point.point);
        
        // 创建线要素
        const lineFeature = new ol.Feature({
          geometry: new ol.geom.LineString(coordinates),
        });
        
        // 添加到路径图层
        this.routeSource.addFeature(lineFeature);
        
        // 计算路径的中心点，用于地图定位
        const extent = lineFeature.getGeometry().getExtent();
        const center = ol.extent.getCenter(extent);
        
        // 移动到路径中心位置
        this.map.getView().animate({
          center: center,
          duration: 500,
          zoom: 10,
        });
        
        // 计算路径距离（粗略估算）
        const distance = this.calculatePathDistance(coordinates);
        
        // 显示成功消息，包含路径距离信息
        alert(`路径规划成功！已按ID从小到大排序并绘制路径。\n路径长度约为: ${distance.toFixed(2)} 公里`);
      } catch (error) {
        console.error('路径规划失败:', error);
        alert('路径规划失败，请重试。');
      } finally {
        // 无论成功与否，都要重置加载状态
        this.isPlanningRoute = false;
      }
    },
    // 新增：计算路径距离（粗略估算）
    calculatePathDistance(coordinates) {
      let totalDistance = 0;
      
      for (let i = 0; i < coordinates.length - 1; i++) {
        // 将投影坐标转换为地理坐标
        const point1 = ol.proj.transform(coordinates[i], 'EPSG:3857', 'EPSG:4326');
        const point2 = ol.proj.transform(coordinates[i + 1], 'EPSG:3857', 'EPSG:4326');
        
        // 计算两点之间的距离（使用Haversine公式）
        const distance = this.haversineDistance(point1[1], point1[0], point2[1], point2[0]);
        totalDistance += distance;
      }
      
      return totalDistance;
    },
    // 新增：Haversine公式计算两点之间的距离
    haversineDistance(lat1, lon1, lat2, lon2) {
      // 地球半径（公里）
      const R = 6371;
      
      // 将经纬度转换为弧度
      const dLat = (lat2 - lat1) * Math.PI / 180;
      const dLon = (lon2 - lon1) * Math.PI / 180;
      
      const a = 
        Math.sin(dLat/2) * Math.sin(dLat/2) +
        Math.cos(lat1 * Math.PI / 180) * Math.cos(lat2 * Math.PI / 180) * 
        Math.sin(dLon/2) * Math.sin(dLon/2);
      
      const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
      const distance = R * c;
      
      return distance;
    }
  },
};
</script>

<style lang="less" scoped>
.home {
  .btns {
    width: 100%;
    height: 200px;
    display: flex;
    justify-content: center;
    align-items: center;
    flex-wrap: wrap;
    button {
      margin: 20px;
    }
  }

  #map {
    width: 100%;
    height: 500px;
    /* 为底部的点信息框留出空间 */
    margin-bottom: 220px;
  }
  // 点信息显示框样式
  .point-info-container {
    position: absolute;
    width: calc(100% - 40px);
    max-height: 200px;
    overflow-y: auto;
    background-color: white;
    padding: 10px;
    border: 1px solid #ccc;
    border-radius: 5px;
    z-index: 1000;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  }
  // 底部位置样式
  .bottom-position {
    bottom: 20px;
    left: 20px;
  }
}
</style>
