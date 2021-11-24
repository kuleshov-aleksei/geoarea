<template>
  <div id="chartdiv"></div>
</template>

<script>
export default {
  name: "Map",
};
import * as am5 from "@amcharts/amcharts5";
import * as am5map from "@amcharts/amcharts5/map";
import am5themes_Animated from "@amcharts/amcharts5/themes/Animated";
import am5geodata_worldLow from "@amcharts/amcharts5-geodata/worldLow";

am5.ready(function () {
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

  polygonSeries.mapPolygons.template.on("active", function (active, target) {
    if (previousPolygon && previousPolygon != target) {
      previousPolygon.set("active", false);
    }
    if (target.get("active")) {
      polygonSeries.zoomToDataItem(target.dataItem);
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
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
#chartdiv {
  width: 100%;
  height: 500px;
}
</style>
