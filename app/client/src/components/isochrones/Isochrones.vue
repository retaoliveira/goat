<template>
  <v-flex xs12 sm8 md4>
    <v-card flat>
      <!-- THEMATIC DATA -->
      <template v-if="isThematicDataVisible === true">
        <v-card-title primary-title class="py-2">
          <v-btn
            text
            class="my-0 py-0"
            icon
            light
            @click="toggleThematicDataVisibility(false)"
          >
            <v-icon color="rgba(0,0,0,0.54)">fas fa-arrow-left</v-icon>
          </v-btn>
          <span class="title">Thematic Data</span>
        </v-card-title>
        <v-card-text class="pr-16 pl-16 pt-0 pb-0 mb-2">
          <v-divider></v-divider>
        </v-card-text>

        <isochrone-thematic-data />
      </template>

      <!-- ISOCHRONE OPTIONS AND RESULTS  -->
      <template v-else>
        <v-subheader>
          <span class="title">{{ $t("isochrones.title") }}</span>
        </v-subheader>

        <v-card-text class="pr-16 pl-16 pt-0 pb-0">
          <v-divider></v-divider>
        </v-card-text>
        <v-card-text>
          <v-layout row>
            <v-flex xs10>
              <v-autocomplete
                solo
                v-model="model"
                :items="items"
                :loading="isLoading"
                label="Search Road"
                :search-input.sync="search"
                item-text="Description"
                append-icon=""
                item-value="API"
                hide-details
                hide-no-data
                prepend-inner-icon="search"
                return-object
              ></v-autocomplete>
            </v-flex>
            <v-flex xs2>
              <v-btn
                class="ml-2 mt-1"
                outlined
                fab
                large
                rounded
                text
                @click="registerMapClick"
              >
                <v-icon color="#30C2FF">fas fa-map-marker-alt</v-icon>
              </v-btn>
            </v-flex>
          </v-layout>
        </v-card-text>
        <v-card-text class="pr-16 pl-16 pt-0 pb-0">
          <v-divider></v-divider>
        </v-card-text>

        <!-- Isochrone Options -->
        <v-subheader
          class="clickable"
          @click="isOptionsElVisible = !isOptionsElVisible"
        >
          <v-icon
            :class="{ activeIcon: isOptionsElVisible, 'mr-2': true }"
            small
            class="mr-2"
            >fas fa-sliders-h</v-icon
          >
          <h3>{{ $t("isochrones.options.title") }}</h3>
        </v-subheader>
        <div v-show="isOptionsElVisible">
          <isochrone-options />
        </div>

        <!-- Isochrone Results  -->
        <v-card-text class="pr-16 pl-16 pt-0 pb-0">
          <v-divider></v-divider>
        </v-card-text>
        <v-subheader
          class="clickable"
          @click="isResultsElVisible = !isResultsElVisible"
        >
          <v-icon
            :class="{
              activeIcon: isResultsElVisible,
              'mr-2': true
            }"
            style="margin-right: 2px;"
            small
            >fas fa-bullseye</v-icon
          >
          <h3>{{ $t("isochrones.results.title") }}</h3>
        </v-subheader>

        <div v-show="isResultsElVisible">
          <isochrone-results />
        </div>
      </template>
      <!-- -- -->
    </v-card>
  </v-flex>
</template>

<script>
import { Mapable } from "../../mixins/Mapable";

//Child components
import IsochroneOptions from "./IsochroneOptions";
import IsochroneResults from "./IsochroneResults";
import IsochronThematicData from "./IsochronesThematicData";

//Store imports
import { mapGetters, mapActions, mapMutations } from "vuex";

//Ol imports
import { transform } from "ol/proj.js";
import VectorSource from "ol/source/Vector";
import VectorLayer from "ol/layer/Vector";
import { Style, Stroke, Fill, Icon } from "ol/style";

export default {
  mixins: [Mapable],
  components: {
    "isochrone-options": IsochroneOptions,
    "isochrone-results": IsochroneResults,
    "isochrone-thematic-data": IsochronThematicData
  },
  data: () => ({
    clicked: false,
    isStartPointElVisible: true,
    isOptionsElVisible: true,
    isResultsElVisible: true,
    //Road Search
    descriptionLimit: 60,
    entries: [],
    model: null,
    search: null,
    isLoading: false
  }),
  computed: {
    ...mapGetters("isochrones", {
      styleData: "styleData",
      isThematicDataVisible: "isThematicDataVisible"
    }),
    ...mapGetters("map", { messages: "messages" }),
    fields() {
      if (!this.model) return [];

      return Object.keys(this.model).map(key => {
        return {
          key,
          value: this.model[key] || "n/a"
        };
      });
    },
    items() {
      return this.entries.map(entry => {
        const Description =
          entry.Description.length > this.descriptionLimit
            ? entry.Description.slice(0, this.descriptionLimit) + "..."
            : entry.Description;

        return Object.assign({}, entry, { Description });
      });
    }
  },
  methods: {
    ...mapActions("isochrones", { calculateIsochrone: "calculateIsochrone" }),
    ...mapMutations("isochrones", {
      addStyleInCache: "ADD_STYLE_IN_CACHE",
      updatePosition: "UPDATE_POSITION",
      addIsochroneLayer: "ADD_ISOCHRONE_LAYER",
      toggleThematicDataVisibility: "TOGGLE_THEMATIC_DATA_VISIBILITY"
    }),

    ...mapMutations("map", {
      startHelpTooltip: "START_HELP_TOOLTIP",
      stopHelpTooltip: "STOP_HELP_TOOLTIP"
    }),
    /**
     * This function is executed, after the map is bound (see mixins/Mapable)
     */
    onMapBound() {
      this.createIsochroneLayer();
    },
    registerMapClick() {
      const me = this;
      me.map.once("singleclick", me.onMapClick);
      me.startHelpTooltip(me.messages.interaction.calculateIsochrone);
      me.map.getTarget().style.cursor = "pointer";
    },
    /**
     * Handler for 'singleclick' on the map.
     * Collects data and passes it to corresponding objects.
     * @param  {ol/MapBrowserEvent} evt The OL event of 'singleclick' on the map
     */
    onMapClick(evt) {
      const me = this;
      //Update Isochrone Position (City or Coordinate)
      const projection = me.map
        .getView()
        .getProjection()
        .getCode();

      const coordinateWgs84 = transform(
        evt.coordinate,
        projection,
        "EPSG:4326"
      );

      me.updatePosition({
        coordinate: coordinateWgs84,
        city: ""
      });
      //Start Isochrone Calculation
      me.calculateIsochrone();
      me.stopHelpTooltip();
      me.map.getTarget().style.cursor = "";
    },

    /**
     * Creates a vector layer for the isochrone calculations results and adds it to the
     * map and store.
     */
    createIsochroneLayer() {
      let me = this;
      let vector = new VectorLayer({
        name: "Isochrone Layer",
        zIndex: 2,
        source: new VectorSource(),
        style: feature => {
          // Style array
          let styles = [];
          let styleData = me.styleData;
          // Get the incomeLevel and modus from the feature properties
          let level = feature.get("step");
          let modus = feature.get("modus");
          let isVisible = feature.get("isVisible");

          let geomType = feature.getGeometry().getType();

          /**
           * Creates styles for isochrone polygon geometry type and isochrone
           * center marker.
           */
          if (
            geomType === "Polygon" ||
            geomType === "MultiPolygon" ||
            geomType === "LineString"
          ) {
            //Check feature isVisible Property
            if (isVisible === false) {
              return;
            }

            //Fallback isochrone style
            if (!modus) {
              if (!styleData.styleCache.default["GenericIsochroneStyle"]) {
                let genericIsochroneStyle = new Style({
                  fill: new Fill({
                    color: [0, 0, 0, 0]
                  }),
                  stroke: new Stroke({
                    color: "#0d0d0d",
                    width: 7
                  })
                });
                let payload = {
                  style: genericIsochroneStyle,
                  isochroneType: "default",
                  styleName: "GenericIsochroneStyle"
                };
                this.addStyleInCache(payload);
              }
              styles.push(
                styleData.styleCache.default["GenericIsochroneStyle"]
              );
            }
            // If the modus is 1 it is a default isochrone
            if (modus === 1 || modus === 3) {
              if (!styleData.styleCache.default[level]) {
                let style = new Style({
                  stroke: new Stroke({
                    color: feature.get("color"),
                    width: 5
                  })
                });
                let payload = {
                  style: style,
                  isochroneType: "default",
                  styleName: level
                };
                this.addStyleInCache(payload);
              }
              styles.push(styleData.styleCache.default[level]);
            } else {
              if (!styleData.styleCache.input[level]) {
                let style = new Style({
                  stroke: new Stroke({
                    color: feature.get("color"),
                    width: 5
                  })
                });
                let payload = {
                  style: style,
                  isochroneType: "input",
                  styleName: level
                };
                this.addStyleInCache(payload);
              }
              styles.push(styleData.styleCache.input[level]);
            }
          } else {
            let path = `img/markers/marker-${feature.get(
              "calculationNumber"
            )}.png`;
            let markerStyle = new Style({
              image: new Icon({
                anchor: [0.5, 0.96],
                src: path,
                scale: 0.5
              })
            });
            styles.push(markerStyle);
          }
          return styles;
        }
      });
      me.map.addLayer(vector);
      this.addIsochroneLayer(vector);
    }
  },
  watch: {
    search(val) {
      console.log(val);

      // Items have already been loaded
      if (this.items.length > 0) return;

      // Items have already been requested
      if (this.isLoading) return;

      this.isLoading = true;

      // Lazily load input items
      fetch("")
        .then(res => res.json())
        .then(res => {
          const { count, entries } = res;
          this.count = count;
          this.entries = entries;
        })
        .catch(err => {
          console.log(err);
        })
        .finally(() => (this.isLoading = false));
    }
  }
};
</script>
<style lang="css" scoped>
.activeIcon {
  color: #30c2ff;
}
</style>