<template>
  <div id="wrapper">
    <div id="chartdiv"></div>
    <div v-if="longSelectedName != null && longSelectedName.length > 0" id="analyze-area">
      <h1>Selected area: {{ longSelectedName }} ({{ shortSelectedId }})</h1>
      <el-button type="primary" @click="startAnalysis">Запустить расчет площади</el-button>
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
      selectedArea: null,
      geometryType: "",
      boundingbox_minX: Number.MAX_SAFE_INTEGER,
      boundingbox_minY: Number.MAX_SAFE_INTEGER,
      boundingbox_maxX: Number.MIN_SAFE_INTEGER,
      boundingbox_maxY: Number.MIN_SAFE_INTEGER,
      boundingPolygon: null,
    };
  },
  methods: {
    handleAreaChanged: function (dataContext) {
      console.log(dataContext);
      this.shortSelectedId = dataContext.id;
      this.longSelectedName = dataContext.name;
      this.selectedArea = dataContext.geometry.coordinates;
      this.geometryType = dataContext.geometryType;
    },
    startAnalysis: function () {
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

      console.log("Min X: " + this.boundingbox_minX + " Min Y: " + this.boundingbox_minY + " Max X: " + this.boundingbox_maxX + " Max Y:" + this.boundingbox_maxY);

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

      this.boundingPolygon = chart.series.push(
        am5map.MapPolygonSeries.new(root, {})
      );

      console.log(am5geodata_worldLow);

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
    }); // end am5.ready()
  },
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
#chartdiv {
  width: 100%;
  height: 500px;
}
</style>
