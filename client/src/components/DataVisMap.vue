<template>
  <div id="mapdiv">
  </div>
</template>

<script>
import L from "leaflet";
import "leaflet-draw";
import "leaflet.markercluster";
import constants from "../constants.js";
import * as d3 from "d3";
import axios from "axios";

export default {
  props: [
    'selectedMapCategory',
    'mapCategories'
  ],
  data() {
    return {
      leafletMap: null,
      selectedStateMarker: null,
      selectedStateCluster: null,
      selectedPolygon: null,
      statePoints: [],
      otherPoints: [],
      stateMarkers: {},
      stateMarkerScale: null,
      otherMarkerScale: null
    };
  },
  methods: {
    /*
     * Initialize the leaflet map and set it as the store's leaflet map
     */
    setupLeafletMap() {
      this.leafletMap = L.map("mapdiv").setView(
          constants.MAP_INIT_CENTER,
          constants.MAP_INIT_ZOOM
      );
      L.tileLayer(constants.MAP_TILE_API_URL, {
        attribution: constants.MAP_TILE_API_ATTR
      }).addTo(this.leafletMap);
      this.leafletMap.zoomControl.setPosition('bottomright');
      this.markerLayerGroup = L.layerGroup().addTo(this.leafletMap);
      this.addLegend();
      this.getStatePoints();
    },
    getStatePoints() {
      axios.get(constants.API_ENDPOINT + 'data-vis/data-map/', {
          headers: {
              Authorization: `${this.$store.getters.authToken}`,
          }
        },).then(response => {
        this.statePoints = response.data;
        this.setStateMarkers();
      });
    },
    getOtherPoints(state_pk) {
      axios.get(constants.API_ENDPOINT + 'data-vis/data-map/?StatePK=' + state_pk, {
          headers: {
              Authorization: `${this.$store.getters.authToken}`,
          }
        }).then(response => {
        this.otherPoints = response.data;
        this.setOtherPoints();
     });
    },
    getPolygon(point) {
      this.clearSelectedPolygon();
      axios.get(constants.API_ENDPOINT + 'data-vis/data-map/polygon/?PolygonID=' + point['PolygonID'], {
          headers: {
              Authorization: `${this.$store.getters.authToken}`,
          }
        }).then(response => {
            this.selectedPolygon = L.geoJSON(response.data, {
              polygonData: point
            });
            this.setPolygon();
            this.selectedPolygon.addTo(this.leafletMap);
          })
    },
    setStateMarkers() {
      this.clearStateMarkers();
      for (let statePoint of this.statePoints) {
        this.stateMarkerScale = d3.scaleLinear()
          .domain([0, statePoint['numAHJs']])
          .range([0, 255]);
        let that = this;
        let colorSaturation = this.stateMarkerScale(this.getCodeNumbersAvg(statePoint));
        this.stateMarkers[statePoint['PolygonID']] = L.marker([statePoint['InternalPLatitude'], statePoint['InternalPLongitude']],{
          markerData: statePoint,
          icon: L.divIcon({ html: '<div style="' + this.getMarkerBackgroundColor([
              255 - colorSaturation,
              colorSaturation
            ]) + '"><span>' + statePoint['numAHJs'] + '</span></div>', className: 'marker-cluster marker-cluster-large', iconSize: new L.Point(40, 40) })
        }).on('click', function (e) {
          let markerData = e.sourceTarget.options.markerData;
          that.getOtherPoints(markerData['PolygonID']);
          if (that.selectedStateMarker) {
            that.leafletMap.addLayer(that.stateMarkers[that.selectedStateMarker]);
          }
          if (that.selectedPolygon) {
            that.selectedPolygon.removeFrom(that.leafletMap);
          }
          that.selectedStateMarker = markerData['PolygonID'];
          that.leafletMap.removeLayer(that.stateMarkers[that.selectedStateMarker]);
        });
        if (this.selectedStateMarker !== statePoint['PolygonID']) {
          this.leafletMap.addLayer(this.stateMarkers[statePoint['PolygonID']]);
        }
      }
    },
    setOtherPoints() {
      this.clearOtherMarkers();
      this.otherMarkerScale = d3.scaleLinear()
          .domain([0, 1])
          .range([0, 255]);
      let that = this;
      this.selectedStateCluster = L.markerClusterGroup({
        chunkedLoading: true,
        iconCreateFunction: function(cluster) {
          let childCount = cluster.getChildCount();
          let mean = d3.mean(cluster.getAllChildMarkers(), d => that.getCodeNumbersAvg(d.options.markerData));
          let colorSaturation = that.otherMarkerScale(mean);
          return L.divIcon({ html: '<div style="' + that.getMarkerBackgroundColor([
              255 - colorSaturation,
              colorSaturation
            ]) + '"><span>' + childCount + '</span></div>', className: 'marker-cluster marker-cluster-medium', iconSize: new L.Point(40, 40) })
        }
      });
      for (let otherPoint of this.otherPoints) {
        let colorSaturation = this.otherMarkerScale(this.getCodeNumbersAvg(otherPoint));
        let icon = otherPoint['AHJPK'] ? '<i class="fas fa-building"></i>' : '<i class="far fa-question-circle"></i>';
        this.selectedStateCluster.addLayer(L.marker([otherPoint['InternalPLatitude'], otherPoint['InternalPLongitude']], {
          markerData: otherPoint,
          icon: L.divIcon({
            className: 'marker-cluster marker-cluster-small',
            html: '<div style="' + this.getMarkerBackgroundColor([
              255 - colorSaturation,
              colorSaturation
            ]) + '"><span>' + icon + '</span></div>'}),
          iconSize: new L.Point(40, 40)
        }).on('click', function (e) {
          let point = e.sourceTarget.options.markerData;
          that.getPolygon(point);
        }).on('mouseover', function (e) {
          let marker = e.sourceTarget;
          marker.openPopup();
        }).on('mouseout', function (e) {
          let marker = e.sourceTarget;
          marker.closePopup();
        }).bindPopup(`<div>${otherPoint['AHJName'] ? otherPoint['AHJName'] : 'No AHJ paired to this polygon'}</div>`));
      }
      this.leafletMap.addLayer(this.selectedStateCluster);
    },
    setPolygon() {
      let colorSaturation = this.otherMarkerScale(this.getCodeNumbersAvg(this.selectedPolygon.options.polygonData));
      let polygonColor = this.getMarkerBackgroundColor([255 - colorSaturation, colorSaturation]);
      polygonColor = polygonColor.substring(polygonColor.indexOf(' ') + 1);
      this.selectedPolygon.setStyle(constants.MAP_PLYGN_CSTM_COLOR(polygonColor));
    },
    clearStateMarkers() {
      Object.keys(this.stateMarkers).forEach(key => { this.leafletMap.removeLayer(this.stateMarkers[key]) });
      this.stateMarkers = {};
    },
    clearOtherMarkers() {
      if (this.selectedStateMarker && this.selectedStateCluster) {
        this.leafletMap.removeLayer(this.selectedStateCluster);
      }
    },
    clearSelectedPolygon() {
      if (this.selectedPolygon) {
        this.leafletMap.removeLayer(this.selectedPolygon);
      }
    },
    getCodeNumbersAvg(point) {
      let ratio = 0;
      if (this.selectedMapCategory === 'all') {
        let numerator = 0;
        let numCategories = 0;
        for (let cat of this.mapCategories) {
          if (cat !== 'all') {
            numerator += point[cat];
            numCategories += 1;
          }
        }
        ratio = numerator / numCategories;
      } else {
        ratio = point[this.selectedMapCategory];
      }
      return ratio;
    },
    getMarkerBackgroundColor(values) {
      for (let i = values.length; i < 4; i++) {
        if (i === 3) {
          values.push(0.6);
        } else {
          values.push(0);
        }
      }
      return 'background-color: rgba(' + values.join(',') + ')';
    },
    addLegend() {
      let legend = L.control({position: 'bottomleft'});
      legend.onAdd = function () {
        let div = L.DomUtil.create('div', 'info legend');
          div.innerHTML +=
            '<div style="text-align: center">Data Coverage</div>' +
            '<br>' +
            '<i></i> ' +
            '<br>' +
            '<span>Less</span><span style="float: right">More</span>';
        return div;
      };

      legend.addTo(this.leafletMap);
    }
  },
  watch: {
    selectedMapCategory: function() {
      this.setStateMarkers();
      this.setOtherPoints();
      if (this.selectedPolygon) {
        this.setPolygon();
      }
    }
  },
  /*
   * Load the leaflet map when this component is mounted
   */
  mounted() {
    this.setupLeafletMap();
  }
};
</script>

<style scoped>

#mapdiv {
  height: 92.5vh;
  width: 100%;
}

::v-deep .info {
  padding: 6px 8px;
  font: 14px/16px Arial, Helvetica, sans-serif;
  background: white;
  background: rgba(255,255,255,0.8);
  box-shadow: 0 0 15px rgba(0,0,0,0.2);
  border-radius: 5px;
}
::v-deep .info h4 {
  margin: 0 0 5px;
  color: #777;
}

::v-deep .legend {
  /*line-height: 1em;*/
  color: #555;
  margin-bottom: 3em;
}
::v-deep .legend i {
  background: linear-gradient(90deg, rgba(255,0,0,1) 0%, rgba(0,255,0,1) 100%);
  width: 128px;
  height: 18px;
  float: left;
  opacity: 0.7;
}

</style>
