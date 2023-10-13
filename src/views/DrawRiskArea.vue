<template>
  <div class="area-editor">
    <div class="area-selector">
      <a-input-search
        @search="onSearch"
        style="width: 100%; margin-bottom: 20px"
        placeholder="请输入区域名称"
      />
      <div
        class="area-selector-buttons"
        style="margin-bottom: 20px; display: flex; align-items: center"
      >
        <a-button type="primary" style="margin-right: 20px" @click="handleEdit"
          >编辑风险区域</a-button
        >
      </div>
      <a-tree :tree-data="treeData" @select="onSelect" :selectedKeys="[selectedKey]" />
    </div>
    <div class="area-map">
      <div class="edit-buttons" style="padding: 10px" v-if="isEdit">
        <span>操作：</span>
        <a-button type="primary" style="margin-right: 20px" @click="handleOk"
          >保存</a-button
        >
        <a-button type="primary" style="margin-right: 20px" @click="handleReset"
          >重置</a-button
        >
        <a-button type="primary" style="margin-right: 20px" @click="handleClear"
          >清除</a-button
        >
        <a-button v-if="isEdit" type="primary" @click="handleUndo">撤销</a-button>
        <br />
      </div>
      <div class="edit-buttons" style="padding: 10px" v-if="isEdit">
        <span>绘制图形：</span>
        <a-button type="primary" style="margin-right: 20px" @click="handlePolygon"
          >多边形</a-button
        >
        <a-button type="primary" style="margin-right: 20px" @click="handleCircle"
          >圆形</a-button
        >
      </div>
      <div id="imageMap" />
    </div>
  </div>
</template>
<script>
import "leaflet/dist/leaflet.css";
import "leaflet/dist/leaflet";
import "leaflet/dist/leaflet-src";
import "leaflet/dist/leaflet-src.esm";
import * as L from "leaflet";
import icon from "leaflet/dist/images/marker-icon.png";
import iconShadow from "leaflet/dist/images/marker-shadow.png";
const DefaultIcon = L.icon({
  iconUrl: icon,
  shadowUrl: iconShadow,
});
L.Marker.prototype.options.icon = DefaultIcon;

export default {
  name: "ImageMap",
  data() {
    return {
      mapUrl: "", // 图片地址
      positions: {}, // 保存每次绘制的图形坐标（分多边形和圆形）
      point: [], // 绘制多边形时，点的每个点坐标
      pol: [], // 绘制多边形时，每个多边形的数据
      tempPolygonFigure: [], // 临时多边形图形
      cir: [], // 绘制圆形时，每个圆的数据
      tempCircleFigure: [], // 临时圆形图形
      newPosition: {}, // 新的点坐标（包含原有的）
      map: null,
      treeData: [
        {
          title: "选矿厂",
          key: "0-0",
          disabled: true,
          children: [
            {
              title: "总平面",
              key: "0-1",
            },
            {
              title: "腰岭沟尾矿库",
              key: "0-2",
            },
          ],
        },
      ],
      areas: [
        {
          key: "0-1",
          label: "总平面",
          color: "#E3170D",
          polygons: [
            [
              [-239, 77],
              [-239, 133],
              [-141, 148],
            ],
            [
              [-101, 309.01289951713034],
              [-99.5, 352.5108990572545],
              [-123, 414.0080708208784],
              [-180, 402.0086226718786],
              [-216, 369.0101402621292],
              [-139, 305.01308346746373],
            ],
          ],
          circles: [],
        },
        {
          key: "0-2",
          label: "腰岭沟尾矿库",
          color: "#0673dd",
          polygons: [
            [
              [-122.5, 222.51687744309038],
              [-120.5, 256.5153138652563],
              [-237.5, 266.5148539894228],
              [-236, 225.01676247413195],
              [-132.5, 221.01694642446546],
            ],
            [
              [-46.046233269598474, 25.046871701034547],
              [-46.046233269598474, 71.04363431627837],
              [-109.10887189292544, 77.04321204870155],
              [-105.60539196940726, 27.546695756210852],
            ],
            [
              [-26.52684512428297, 275.0292772186642],
              [-26.026347992351816, 349.02406925188257],
              [-73.07307839388145, 311.5267084242381],
            ],
          ],
          circles: [
            {
              point: [-39.5, 174.51908484709134],
              radius: 23.113204034300395,
            },
            {
              radius: 27.966066923399133,
              point: [-290.96246620026614, 204.11798049501817],
            },
          ],
        },
      ],
      areaList: [], // 保存所有的图形，保存key值(服务区获取)
      isEdit: false, // 是否进入编辑模式
      selectedKey: "", // 选择的区域key值
      markerLayers: [], //标记数组
      markerLayerGroup: null, // 标记组
      lineLayers: [], // 线条数组
      lineLayerGroup: null, // 线条组

      i: null,
      radius: 0,
      tempCircle: null,
    };
  },
  mounted() {
    this.initDate();
  },
  methods: {
    // 求多边形的中心点
    getCenterPoint(points) {
      var sumX = 0;
      var sumY = 0;
      for (let i = 0; i < points.length; i++) {
        sumX += points[i][0];
        sumY += points[i][1];
      }
      return [sumX / points.length, sumY / points.length];
    },
    loadAreas() {
      // 后面从服务器拿到所有划分区域的信息，显示图形
      // 调用位置:初始化,完成编辑,重置

      var list = [];
      // TODO:从服务器拿this.areas需要的数据
      for (let area of this.areas) {
        var polygon = null;
        var circle = null;
        var layer = null;
        var textarea = [];
        console.log(area);

        var textIcon = L.divIcon({
          html: area.label,
          iconSize: 30,
          iconAnchor: [32, 10],
          className: "text-icon",
        });
        if (area.polygons.length != 0) {
          for (let i = 0; i < area.polygons.length; i++) {
            polygon = L.polygon(area.polygons[i], { color: area.color, fillOpacity: 0.7 })
              .addTo(this.map)
              .bindPopup(area.label);
            layer = L.polygon(area.polygons[i], { color: area.color });

            // 文字标注
            var points = area.polygons[i];
            var text = L.marker(this.getCenterPoint(points), { icon: textIcon }).addTo(
              this.map
            );
            textarea.push(text);

            var obj = {
              key: area.key,
              type: "polygon",
              figure: polygon,
              layer: layer,
              label: area.label,
              pos: area.polygons[i],
              color: area.color,
              textarea: textarea,
            };
            list.push(obj);
          }
        }
        if (area.circles.length != 0) {
          for (let i = 0; i < area.circles.length; i++) {
            circle = L.circle(area.circles[i].point, {
              radius: area.circles[i].radius,
              color: area.color,
              fillOpacity: 0.7,
            })
              .addTo(this.map)
              .bindPopup(area.label);
            layer = L.circle(area.circles[i].point, {
              radius: area.circles[i].radius,
              color: area.color,
            });

            // 文字标注

            var text = L.marker(area.circles[i].point, { icon: textIcon }).addTo(
              this.map
            );
            textarea.push(text);

            var obj = {
              key: area.key,
              type: "circle",
              figure: circle,
              // layer: layer,
              label: area.label,
              pos: area.circles[i],
              color: area.color,
              textarea: textarea,
            };
            list.push(obj);
          }
        }
        this.areaList = list;
        console.log("this.areaList", this.areaList);
      }
    },
    initDate() {
      if (this.map) {
        this.map.remove();
      }
      // TODO：服务器获取图片
      this.mapUrl = require("../img/test2.jpg");
      this.map = L.map("imageMap", {
        attributionControl: false, // 右下角logo
        minZoom: 1,
        maxZoom: 4,
        center: [0, 0],
        zoom: 1,
        crs: L.CRS.Simple, // 这表明leaflet使用1：1映射，在屏幕像素和经纬度坐标系统之间。换句话说，我们的图片是平的，不是全球的，我们在投影一张平面图片。
      });
      // 定义了图片尺寸和它的路径，路径可以引用网络链接
      var w = 4000;
      var h = 3000;
      var url = this.mapUrl;
      // 把图片通过地图的方式放出来，所以需要一个像素坐标到经纬度坐标（系统）的转换。 把西南，东北角的像素坐标逆映射为经纬度坐标网。
      var southWest = this.map.unproject([0, h], this.map.getMaxZoom() - 1);
      var northEast = this.map.unproject([w, 0], this.map.getMaxZoom() - 1);
      var bounds = new L.LatLngBounds(southWest, northEast);
      L.imageOverlay(url, bounds).addTo(this.map);

      this.cir = [];
      this.pol = [];
      this.tempPolygonFigure = [];
      this.tempCircleFigure = [];

      this.loadAreas();

      this.map.setMaxBounds(bounds);
    },

    handlePolygon() {
      if (this.positions.circle) {
        this.$message.warning("请先保存！");
      } else {
        this.map.off("mousedown");
        this.map.off("mouseup");
        this.map.off("mousemove");

        this.map.doubleClickZoom.disable(); //禁止双击放大地图

        this.map.on("click", this.drawPolygon);
        let that = this;
        this.map.on("dblclick", this.handleDbclick);
      }
    },
    handleDbclick(e) {
      var p = L.polygon(this.point, { color: "#FFD700", fillOpacity: 0.7 }).addTo(
        this.map
      );
      this.tempPolygonFigure.push(p);
      this.pol.push(this.point);
      this.$set(this.positions, "polygon", this.pol); // 保存点坐标
      this.clearDraw();
      this.point = [];
      console.log("双击啦");
    },
    handleCircle() {
      // this.handleReset();
      if (this.positions.polygon) {
        this.$message.warning("请先保存！");
      } else {
        var map = this.map;
        var r = 0;
        var i = null;
        var tempCircle = new L.circle();
        let that = this;
        map.dragging.disable(); //将mousemove事件移动地图禁用
        map.off("click");
        map.on("mousedown", onmouseDown);
        map.on("mouseup", onmouseUp);
        map.on("mousemove", onMove);
        function onmouseDown(e) {
          i = e.latlng;
          map.dragging.disable(); //将mousemove事件移动地图禁用
          console.log("圆点", i);
          //确定圆心
        }
        function onMove(e) {
          if (i) {
            r = Math.sqrt(
              Math.pow(e.latlng.lng - i.lng, 2) + Math.pow(e.latlng.lat - i.lat, 2)
            );
            console.log("半径", r);
            tempCircle.setLatLng(i);
            tempCircle.setRadius(r);
            tempCircle.setStyle({ color: "#FFD700", fillOpacity: 0.7 });
            map.addLayer(tempCircle);
          }
        }
        function onmouseUp(e) {
          tempCircle.remove();
          r = Math.sqrt(
            Math.pow(e.latlng.lng - i.lng, 2) + Math.pow(e.latlng.lat - i.lat, 2)
          );
          var c = L.circle(i, {
            radius: r,
            color: "#FFD700",
            fillOpacity: 0.7,
          }).addTo(map);
          map.dragging.enable();
          var obj = {
            point: [i.lat, i.lng],
            radius: r,
          };
          that.tempCircleFigure.push(c);
          that.cir.push(obj);
          that.$set(that.positions, "circle", that.cir);
          i = null;
          r = 0;
        }
      }
    },
    // 画多边形
    drawPolygon(e) {
      console.log("You clicked the map at ", e.latlng);
      var latlng = e.latlng;
      var point = [latlng.lat, latlng.lng]; // 当前点击的坐标

      // 标记显示
      var myIcon = L.icon({
        iconUrl: require("../assets/location.png"),
        iconSize: [40, 40],
        iconAnchor: [20, 40],
        popupAnchor: [-3, -76],
      });
      var markerLayer = new L.marker(point, { icon: myIcon });
      this.markerLayers.push(markerLayer);
      this.markerLayerGroup = L.layerGroup(this.markerLayers);
      this.map.addLayer(this.markerLayerGroup);

      // 连线
      this.point.push(point);

      var lineLayer = new L.polyline(this.point, { color: "#FFD700",  fillOpacity: 0.7});
      this.lineLayers.push(lineLayer);
      this.lineLayerGroup = L.layerGroup(this.lineLayers);
      this.map.addLayer(this.lineLayerGroup);
    },
    handleEdit() {
      if (this.selectedKey.length !== 0) {
        this.isEdit = true;
        this.positions = {};
        this.point = [];
      } else {
        this.$message.warning("请选择区域！");
      }
    },
    // 显示选择区域
    selectArea(key) {
      this.newPosition = {};
      var _pos1 = [];
      var _pos2 = [];
      for (let area of this.areaList) {
        if (area.key === key) {
          area.figure.setStyle({
            color: "#FFD700",
          });
          if (area.type === "circle") {
            _pos1.push(area.pos);
            this.$set(this.newPosition, "circles", _pos1);
          } else {
            _pos2.push(area.pos);
            this.$set(this.newPosition, "polygons", _pos2);
          }
          console.log("this.newPosition", this.newPosition);
        } else {
          area.figure.setStyle({
            color: area.color,
          });
        }
      }
    },
    onSelect(selectedKeys) {
      // 如果编辑的时候点击其他区域，会跳转至该区域的查看状态且不会保存编辑的内容
      if (this.isEdit) {
        this.isEdit = false;
        this.clearDraw();
        this.initDate();
      }
      if (selectedKeys.length != 0) {
        console.log("selectedKeys", selectedKeys[0]);
        this.selectedKey = selectedKeys[0];
        this.selectArea(this.selectedKey);
      } else {
        this.selectedKey = "";
        this.isEdit = false;
        this.handleReset();
      }
    },
    onSearch(value) {
      console.log(value);
    },
    // 保存已绘制的图形
    saveFigure() {
      if (this.positions.polygon) {
        if (this.newPosition.polygons) {
          for (let item of this.positions.polygon) {
            this.newPosition.polygons.push(item);
          }
        } else {
          this.newPosition.polygons = this.positions.polygon;
        }
      }

      if (this.positions.circle) {
        if (this.newPosition.circles) {
          for (let item of this.positions.circle) {
            this.newPosition.circles.push(item);
          }
        } else {
          this.newPosition.circles = this.positions.circle;
        }
      }
      this.positions = {};
      this.point = [];
    },
    // 完成：上传绘制的点坐标，重新加载地图并保留选中的区域
    handleOk() {
      this.isEdit = false;
      // 清空标记和线
      if (this.markerLayerGroup && this.lineLayerGroup) {
        this.markerLayerGroup.clearLayers();
        this.lineLayerGroup.clearLayers();
      }
      this.markerLayers = [];
      this.lineLayers = [];
      console.log("this.positions", this.positions);
      this.saveFigure();
      console.log("this.newPosition", this.newPosition);
      // // TODO：后面不需要画，只需要将this.newPosition和key值上传到后端，再刷新一下界面即可
      // L.polygon(this.positions.polygon, { color: "yellow" }).addTo(this.map);
      // L.circle(this.positions.circles.point, {radius: this.positions.circles.radius, color: "yellow" }).addTo(this.map);
      // this.selectedKey = "";
      this.initDate();
      this.onSelect([this.selectedKey]);
    },
    // 撤销
    handleUndo() {
      // 多边形的边撤销
      if (this.markerLayers.length != 0 && this.lineLayers.length != 0) {
        this.markerLayers.pop();
        this.markerLayerGroup.clearLayers();
        this.markerLayerGroup = L.layerGroup(this.markerLayers);
        this.map.addLayer(this.markerLayerGroup);

        this.lineLayers.pop();
        this.lineLayerGroup.clearLayers();
        this.lineLayerGroup = L.layerGroup(this.lineLayers);
        this.map.addLayer(this.lineLayerGroup);

        this.point.pop();
      }
      // 多边形撤销
      else if (this.pol.length != 0) {
        this.pol.pop();
        this.tempPolygonFigure.at(-1).remove();
        this.tempPolygonFigure.pop();
      }
      // 撤销圆形
      else if (this.cir.length != 0) {
        this.cir.pop();
        this.tempCircleFigure.at(-1).remove();
        this.tempCircleFigure.pop();
      }
    },
    // 清除正在绘制的临时图形
    clearDraw() {
      if (this.lineLayerGroup && this.markerLayerGroup) {
        this.lineLayerGroup.clearLayers();
        this.markerLayerGroup.clearLayers();
      }
      this.lineLayers = [];
      this.markerLayers = [];
    },
    // 重置：清除绘制的图形，重新加载地图并保留选中的区域
    handleReset() {
      this.clearDraw();
      this.positions = {};
      this.point = [];
      this.initDate();
      this.selectArea(this.selectedKey);
    },
    // 清除：清除绘制的图形和初始的图形
    handleClear() {
      this.clearDraw();
      for (let area of this.areaList) {
        console.log("1111", area);
        if (this.selectedKey === area.key) {
          area.figure.remove();
          for (let text of area.textarea) {
            text.remove();
          }
        }
      }
      for (let temp of this.tempCircleFigure) {
        temp.remove();
      }
      for (let temp of this.tempPolygonFigure) {
        temp.remove();
      }
      this.positions = {};
      this.newPosition.polygons = [];
      this.newPosition.circles = [];
      console.log("handleClear", this.newPosition);
    },
  },
};
</script>
<style scope>
.area-editor {
  width: 100%;
  height: 100%;
  display: flex;
}
.area-selector {
  width: 30%;
  padding: 30px;
}
.area-map {
  width: 70%;
}
#imageMap {
  width: 100%;
  height: 90%;
  border: 1px solid #ccc;
  margin-bottom: 10px;
}
.text-icon {
  background: transparent;
  font-size: 14px;
  color: #000000;
  font-weight: 1000;
  white-space: nowrap;
  margin-left: -33px;
}
</style>
