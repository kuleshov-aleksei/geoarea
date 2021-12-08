<template>
  <div id="wrapper">
    <div id="chartdiv"></div>
    <div id="controls">
      <div><el-button class="control-element" type="primary" @click="foundBoundingBox">Найти bounding-box</el-button></div>
      <div><el-input class="control-element" placeholder="Введите количество точек" v-model="targetPointsCount" clearable></el-input></div>
      <div><el-button class="control-element" type="primary" @click="generatePoints">Начать рассчет площади</el-button></div>
      <div><el-button class="control-element" type="primary" @click="resetState">Сбросить состояние</el-button></div>
      <!--<div><el-button class="control-element" type="primary" @click="doSmth">Тестовая кнопка</el-button></div>-->
    </div>
    <div id="selected-area">
      <h1>Выбранная область: {{ longSelectedName }} ({{ shortSelectedId }})</h1>
      <div v-if="targetPointsCount > 1000">В целях оптимизации, количество точек на рисунке ограничено 1000 точками. Остальные точки просчитываются в фоне</div>
    </div>
    <div id="debug-info">
      <div v-if="boundingbox_minX != 0">
        Границы: [{{ boundingbox_minX }} ; {{ boundingbox_minY }}] [{{ boundingbox_maxX }} ; {{ boundingbox_maxY }}] 
        <div id="selected-area"></div>
      </div>
      <div class="basic-div-info">Точек в области: {{ calculated.pointsInsideArea }} (из {{ calculated.pointsGenerated }} сгенерированных)</div>
      <div class="basic-div-info">Ширина области: {{ calculated.width.toFixed(2) }} км</div>
      <div class="basic-div-info">Высота области: {{ calculated.height.toFixed(2) }} км</div>
      <div class="basic-div-info">Площадь объекта: <b>{{ calculated.area.toFixed(2) }} км2</b></div>
    </div>
  </div>
</template>

<script>
import * as am5 from "@amcharts/amcharts5";
import * as am5map from "@amcharts/amcharts5/map";
import am5themes_Animated from "@amcharts/amcharts5/themes/Animated";
import am5geodata_worldLow from "@amcharts/amcharts5-geodata/worldLow";
import * as GeoJsonGeometriesLookup from "geojson-geometries-lookup";


export default {
  name: "Map",
  data() {
    return {
      shortSelectedId: "",
      longSelectedName: "",
      //selectedArea: null,
      //geometryType: "",
      boundingbox_minX: 0,
      boundingbox_minY: 0,
      boundingbox_maxX: 0,
      boundingbox_maxY: 0,
      targetPointsCount: 100,
      //boundingPolygon: null,
      calculated: {
        pointsInsideArea: 0,
        pointsGenerated: 0,
        width: 0,
        height: 0,
        area: 0,
        coef: 1.04
      }
    };
  },
  methods: {
    handleAreaChanged: function (dataContext) {
      //console.log(dataContext);
      this.shortSelectedId = dataContext.id;
      this.longSelectedName = dataContext.name;
      this.selectedArea = dataContext.geometry.coordinates;
      this.geometryType = dataContext.geometryType;
      
      this.drawSelectedArea(this.shortSelectedId);
    },
    doSmth: function () {
      
    },
    generatePoints: function () {
      this.points = [];
      this.pointSeries.show();
      this.calculated.pointsInsideArea = 0;
      this.calculated.pointsGenerated = 0;

      //for (let i = 0; i < this.targetPointsCount; i++) {
      //  processIteration(i);
      //}

      this.index = 0;
      
      this.doChunk(this.targetPointsCount); 

      while (this.points.length < 1000)
      {
        this.points.push({ long: 0, lat: 0 });
      }
      
      this.pointSeries.data.setAll(this.points);
      this.pointSeries.toFront();
    },
    processIteration: function(index) {
      let x = this.getRandomArbitrary(this.boundingbox_minX, this.boundingbox_maxX);
      let y = this.getRandomArbitrary(this.boundingbox_minY, this.boundingbox_maxY);

      if (index < 1000)
      {
        this.points.push({ long: x, lat: y });
      }

      if (this.isInsideArea(x, y))
      {
        this.calculated.pointsInsideArea++;
      }

      this.calculated.pointsGenerated++;
    },
    doChunk: function() {
        var cnt = 10000;
        while (cnt-- && this.index < this.targetPointsCount) {
            this.processIteration(this.index);
            ++this.index;
        }
        //if (this.index >= this.targetPointsCount) {
          this.calcArea();
        //}
        if (this.index < this.targetPointsCount) {
            setTimeout(this.doChunk, 1);
        }
    },
    calcArea: function() {
      this.calculated.width = this.calcCrow(this.boundingbox_minX, this.boundingbox_minY, this.boundingbox_maxX, this.boundingbox_minY);
      this.calculated.height = this.calcCrow(this.boundingbox_minX, this.boundingbox_minY, this.boundingbox_minX, this.boundingbox_maxY);
      this.calculated.area = (this.calculated.pointsInsideArea / this.calculated.pointsGenerated) * this.calculated.width * this.calculated.height * this.calculated.coef;
    },
    calcCrow: function(lat1, lon1, lat2, lon2) {
      var R = 6371; // km
      var dLat = this.toRad(lat2-lat1);
      var dLon = this.toRad(lon2-lon1);
      lat1 = this.toRad(lat1);
      lat2 = this.toRad(lat2);

      var a = Math.sin(dLat/2) * Math.sin(dLat/2) +
        Math.sin(dLon/2) * Math.sin(dLon/2) * Math.cos(lat1) * Math.cos(lat2); 
      var c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a)); 
      var d = R * c;
      return d;
    },
    toRad: function(Value) 
    {
      return Value * Math.PI / 180;
    },
    isInsideArea: function(x, y) {
      const glookup = new GeoJsonGeometriesLookup(this.geoChunkArea);
      const point1 = {type: "Point", coordinates: [x, y]}
      return glookup.countContainers(point1) !== 0;
    },
    foundBoundingBox: function () {
      this.resetState();
      //console.log(this.selectedArea);
      if (this.geometryType === "Polygon") {
        let [minX, minY, maxX, maxY] = this.findMinMaxPolygon(
          this.selectedArea
        );
        this.boundingbox_minX = minX;
        this.boundingbox_minY = minY;
        this.boundingbox_maxX = maxX;
        this.boundingbox_maxY = maxY;
      } else if (this.geometryType === "MultiPolygon") {
        let [minX, minY, maxX, maxY] = this.findMinMaxMultiPolygon(
          this.selectedArea
        );
        this.boundingbox_minX = minX;
        this.boundingbox_minY = minY;
        this.boundingbox_maxX = maxX;
        this.boundingbox_maxY = maxY;
      } else {
        console.log("unknown geometry type");
        return;
      }

      //console.log("Min X: " + this.boundingbox_minX + " Min Y: " + this.boundingbox_minY + " Max X: " + this.boundingbox_maxX + " Max Y:" + this.boundingbox_maxY);

      this.boundingPolygon.data.clear();
      this.boundingPolygon.data.push({
        geometry: {
          type: "Polygon",
          coordinates: [
            [
              [this.boundingbox_minX, this.boundingbox_minY],
              [this.boundingbox_minX, this.boundingbox_maxY],
              [this.boundingbox_maxX, this.boundingbox_maxY],
              [this.boundingbox_maxX, this.boundingbox_minY],
              [this.boundingbox_minX, this.boundingbox_minY],
            ]
          ]
        }
      });
      this.boundingPolygon.show();

      this.drawSelectedArea(this.shortSelectedId);
    },
    findMinMaxMultiPolygon: function (area) {
      var minX = Number.MAX_SAFE_INTEGER;
      var minY = Number.MAX_SAFE_INTEGER;
      var maxX = Number.MIN_SAFE_INTEGER;
      var maxY = Number.MIN_SAFE_INTEGER;

      for (const polygon of area) {
        let [localMinX, localMinY, localMaxX, localMaxY] =
          this.findMinMaxPolygon(polygon);
        if (localMinX < minX) {
          minX = localMinX;
        }
        if (localMaxX > maxX) {
          maxX = localMaxX;
        }

        if (localMinY < minY) {
          minY = localMinY;
        }
        if (localMaxY > maxY) {
          maxY = localMaxY;
        }
      }

      return [minX, minY, maxX, maxY];
    },
    findMinMaxPolygon: function (globalPolygon) {
      var minX = Number.MAX_SAFE_INTEGER;
      var minY = Number.MAX_SAFE_INTEGER;
      var maxX = Number.MIN_SAFE_INTEGER;
      var maxY = Number.MIN_SAFE_INTEGER;

      this.m_j = 0;
      for (const polygon of globalPolygon) {
        for (const point of polygon) {
          if (point.length == 2) {
            let x = point[0];
            let y = point[1];

            if (x < minX) {
              minX = x;
            } else if (x > maxX) {
              maxX = x;
            }

            if (y < minY) {
              minY = y;
            } else if (y > maxY) {
              maxY = y;
            }
          }
        }
      }

      return [minX, minY, maxX, maxY];
    },
    resetState: function() {
      this.boundingbox_minX = Number.MAX_SAFE_INTEGER;
      this.boundingbox_minY = Number.MAX_SAFE_INTEGER;
      this.boundingbox_maxX = Number.MIN_SAFE_INTEGER;
      this.boundingbox_maxY = Number.MIN_SAFE_INTEGER;
      this.calculated.pointsInsideArea = 0;
      this.calculated.pointsGenerated = 0;
      this.calculated.width = 0;
      this.calculated.height = 0;
      this.calculated.area = 0;
      this.boundingPolygon.hide();
      this.pointSeries.hide();
    },
    drawSelectedArea: function(region) {
      if (this.selectedPolygonSeries != null) {
        this.selectedAreaChart.series.removeIndex(this.selectedAreaChart.series.indexOf(this.selectedPolygonSeries));
        this.selectedPolygonSeries.dispose();
      }
      this.selectedPolygonSeries = this.selectedAreaChart.series.push(
        am5map.MapPolygonSeries.new(this.selectedAreaRoot, {
          geoJSON: am5geodata_worldLow,
          include: [region],
        })
      );

      var geoChunk = am5geodata_worldLow.features.filter(item => item.id === region);
      this.geoChunkArea = {
        type: "FeatureCollection",
        features: geoChunk
      };

      if (this.boundingPolygon == null) {
        this.boundingPolygon = this.selectedAreaChart.series.push(
          am5map.MapPolygonSeries.new(this.selectedAreaRoot, {
            fill: am5.color(0xffaaaa),
          })
        );
      }

      if (this.pointSeries == null) {
        console.log("Creating point series");
        this.pointSeries = this.selectedAreaChart.series.push(
          am5map.MapPointSeries.new(this.selectedAreaRoot, {
              latitudeField: "lat",
              longitudeField: "long"
          })
        );

        this.pointSeries.bullets.push(() => {
          return am5.Bullet.new(this.selectedAreaRoot, {
            sprite: am5.Circle.new(this.selectedAreaRoot, {
              radius: 5,
              fill: am5.color(0x00ff00)
            })
          });
        });
      }

      for (let i = 0; i < 1000; i++) {
          this.pointSeries.data.push({
          //geometry: { type: "Point", coordinates: [0, 0] },
          geometry: { type: "Point", coordinates: [45.2666, -20.5631] },
          title: "prefill" + i
        });
      }

      this.selectedPolygonSeries.mapPolygons.template.setAll({
        tooltipText: "{name}",
        toggleKey: "active",
        interactive: true,
      });

      // Make stuff animate on load
      this.selectedAreaChart.appear(1000, 100);
    },
    getRandomArbitrary: function(min, max) {
        return Math.random() * (max - min) + min;
    }
  },
  mounted() {
    am5.ready(() => {
      // Create root element
      // https://www.amcharts.com/docs/v5/getting-started/#Root_element
      var root = am5.Root.new("chartdiv");

      // Set themes
      // https://www.amcharts.com/docs/v5/concepts/themes/
      root.setThemes([am5themes_Animated.new(root)]);

      // Create the map chart
      // https://www.amcharts.com/docs/v5/charts/map-chart/
      var chart = root.container.children.push(
        am5map.MapChart.new(root, {
          panX: "translateX",
          panY: "translateY",
          projection: am5map.geoMercator(),
        })
      );

      // Create main polygon series for countries
      // https://www.amcharts.com/docs/v5/charts/map-chart/map-polygon-series/
      var polygonSeries = chart.series.push(
        am5map.MapPolygonSeries.new(root, {
          geoJSON: am5geodata_worldLow,
          exclude: ["AQ"],
        })
      );

      //console.log(am5geodata_worldLow);

      polygonSeries.mapPolygons.template.setAll({
        tooltipText: "{name}",
        toggleKey: "active",
        interactive: true,
      });

      polygonSeries.mapPolygons.template.states.create("hover", {
        fill: root.interfaceColors.get("primaryButtonHover"),
      });

      polygonSeries.mapPolygons.template.states.create("active", {
        fill: root.interfaceColors.get("primaryButtonHover"),
      });

      var previousPolygon;

      polygonSeries.mapPolygons.template.on("active", (_, target) => {
        if (previousPolygon && previousPolygon != target) {
          previousPolygon.set("active", false);
        }
        if (target.get("active")) {
          polygonSeries.zoomToDataItem(target.dataItem);
          this.handleAreaChanged(target.dataItem.dataContext);
        } else {
          chart.goHome();
        }
        previousPolygon = target;
      });

      // Add zoom control
      // https://www.amcharts.com/docs/v5/charts/map-chart/map-pan-zoom/#Zoom_control
      chart.set("zoomControl", am5map.ZoomControl.new(root, {}));

      // Set clicking on "water" to zoom out
      chart.chartContainer.get("background").events.on("click", function () {
        chart.goHome();
      });

      // Make stuff animate on load
      chart.appear(1000, 100);

      // Prepare selected area
      this.selectedAreaRoot = am5.Root.new("selected-area");
      this.selectedAreaChart = this.selectedAreaRoot.container.children.push(
        am5map.MapChart.new(this.selectedAreaRoot, {
          panX: "translateX",
          panY: "translateY",
          projection: am5map.geoMercator(),
        })
      );

    }); // end am5.ready()
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
#wrapper {
  display: grid; 
  grid-template-columns: 1fr 1fr 1fr; 
  grid-template-rows: 500px auto; 
  gap: 0px 0px; 
  grid-template-areas: 
    "chartdiv chartdiv selected-area"
    "controls debug-info ."; 
  height: 900px;
  width: 100%;
}

#chartdiv {
  width: 100%;
  height: 500px;
  grid-area: chartdiv;
}
#selected-area {
  width: auto;
  grid-area: selected-area;
}
#controls { 
  grid-area: controls;
}
#debug-info {
  grid-area: debug-info;
}
.control-element {
  width: 300px;
  margin: 4px;
}
.basic-div-info {
  padding: 5px;
}
</style>
