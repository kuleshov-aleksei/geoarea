<template>
  <div id="wrapper">
    <div id="chartdiv"></div>
    <div id="controls">
      <el-button type="primary" @click="foundBoundingBox">Найти bounding-box</el-button>
    </div>
    <div id="selected-area">
      <h1>Выбранная область: {{ longSelectedName }} ({{ shortSelectedId }})</h1>
    </div>
    <div id="debug-info">
      <div v-if="boundingbox_minX != 0">
        Границы: [{{ boundingbox_minX }} ; {{ boundingbox_minY }}] [{{ boundingbox_maxX }} ; {{ boundingbox_maxY }}] 
        <div id="selected-area"></div>
      </div>
    </div>
  </div>
</template>

<script>
import * as am5 from "@amcharts/amcharts5";
import * as am5map from "@amcharts/amcharts5/map";
import am5themes_Animated from "@amcharts/amcharts5/themes/Animated";
import am5geodata_worldLow from "@amcharts/amcharts5-geodata/worldLow";

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
      //boundingPolygon: null,
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
    },
    drawSelectedArea: function(region) {
      //this.selectedAreaChart.series.clear();
      if (this.selectedPolygonSeries != null) {
        this.selectedAreaChart.series.removeIndex(this.selectedAreaChart.series.indexOf(this.selectedPolygonSeries));
        //this.selectedAreaChart.series.removeIndex(this.selectedAreaChart.series.indexOf(this.boundingPolygon));
        this.selectedPolygonSeries.dispose();
      }
      this.selectedPolygonSeries = this.selectedAreaChart.series.push(
        am5map.MapPolygonSeries.new(this.selectedAreaRoot, {
          geoJSON: am5geodata_worldLow,
          include: [region],
        })
      );

      if (this.boundingPolygon == null) {
        this.boundingPolygon = this.selectedAreaChart.series.push(
          am5map.MapPolygonSeries.new(this.selectedAreaRoot, {
            fill: am5.color(0xffaaaa),
          })
        );
      }

      this.selectedPolygonSeries.mapPolygons.template.setAll({
        tooltipText: "{name}",
        toggleKey: "active",
        interactive: true,
      });

      // Make stuff animate on load
      this.selectedAreaChart.appear(1000, 100);
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

</style>
