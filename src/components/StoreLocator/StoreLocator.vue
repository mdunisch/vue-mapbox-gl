<template>
  <div :class="classes.root || {}">
    <div :class="(classes.region || {}).map || {}">
      <template v-if="isLoading">
        <transition v-bind="(transitions.loader || {}).map || {}">
          <!-- @slot Use this slot to define a custom loader for the map region. -->
          <slot name="map-loader">
            <transition v-bind="(transitions.loader || {}).default || {}">
              <!-- @slot Use this slot to define a custom loader. -->
              <slot name="loader">
                Loading...
              </slot>
            </transition>
          </slot>
        </transition>
      </template>

      <!-- @slot Use this slot to display information before the map. -->
      <slot name="before-map" />

      <mapbox-map
        :class="classes.map || {}"
        v-bind="{ ...mapboxMap, accessToken }"
        @mb-created="onMapCreated"
        @mb-movestart="onMapMovestart"
        @mb-moveend="onMapMoveend"
        @mb-load="onMapLoad">
        <mapbox-cluster
          v-bind="{ ...mapboxCluster, data: filteredGeoJson }"
          @mb-feature-click="onClusterFeatureClick"
          @mb-feature-mouseenter="onClusterFeatureMouseenter"
          @mb-feature-mouseleave="onClusterFeatureMouseleave"
          @mb-cluster-click="onClusterClusterClick" />
        <!--
          @slot Use this slot to add components from @studiometa/vue-mapbox-gl to the map.
            @binding {Object}  map             The map instance.
            @binding {GeoJSON} geojson         The GeoJSON used for the cluster.
            @binding {GeoJSON} filteredGeoJson The filtered GeoJSON.
            @binding {Array}   items           The list of items.
            @binding {Array}   filteredItems   The filtered list of items.
            @binding {Object}  selectedItem    The selected item.
        -->
        <slot
          name="map"
          :map="map"
          :geojson="geoJson"
          :filtered-geojson="filteredGeoJson"
          :items="items"
          :filtered-items="filteredItems"
          :selected-item="selectedItem" />
      </mapbox-map>

      <!-- @slot Use this slot to display information after the map. -->
      <slot name="after-map" />
    </div>
    <div :class="(classes.region || {}).search || {}">
      <template v-if="isLoading">
        <transition v-bind="(transitions.loader || {}).search || {}">
          <!-- @slot Use this slot to define a custom loader for the search region. -->
          <slot name="search-loader">
            <transition v-bind="(transitions.loader || {}).default || {}">
              <!-- @slot Use this slot to define a custom loader. -->
              <slot name="loader">
                Loading...
              </slot>
            </transition>
          </slot>
        </transition>
      </template>

      <!-- @slot Use this slot to display information before the search. -->
      <slot
        name="before-search"
        :items="items"
        :filtered-items="filteredItems"
        :selected-item="selectedItem" />

      <mapbox-geocoder
        :class="classes.search || {}"
        v-bind="{ ...mapboxGeocoder, accessToken }"
        @mb-result="onGeocoderResult"
        @mb-created="onGeocoderCreated" />

      <!-- @slot Use this slot to display information after the search. -->
      <slot
        name="after-search"
        :items="items"
        :filtered-items="filteredItems"
        :selected-item="selectedItem" />
    </div>
    <div :class="(classes.region || {}).list || {}">
      <template v-if="isLoading || listIsLoading">
        <transition v-bind="(transitions.loader || {}).list || {}">
          <!-- @slot Use this slot to define a custom loader for the list region. -->
          <slot name="list-loader">
            <transition v-bind="(transitions.loader || {}).default || {}">
              <!-- @slot Use this slot to define a custom loader. -->
              <slot name="loader">
                Loading...
              </slot>
            </transition>
          </slot>
        </transition>
      </template>
      <template v-else>
        <!--
          @slot Use this slot to display information before the list.
            @binding {Array} items         The full list of items.
            @binding {Array} filteredItems The filtered list of items.
        -->
        <slot
          name="before-list"
          :items="items"
          :filtered-items="filteredItems"
          :selected-item="selectedItem">
          <p>Result(s): {{ filteredItems.length.toFixed(0) }}</p>
        </slot>

        <vue-scroller v-if="filteredItems.length > 0">
          <ul :class="classes.list || {}">
            <li
              v-for="(item, index) in filteredItems"
              :key="item.id"
              :class="classes.listItem || {}"
              @click="onListItemClick(item)">
              <!--
                @slot Use this slot to customize the display of the list items.
                  @binding {Object} item          An item.
                  @binding {Object} selected-item The currently selected item.
              -->
              <slot
                name="list-item"
                :item="item"
                :index="index"
                :selected-item="selectedItem">
                Lat: {{ item.lat }} <br>
                Lng: {{ item.lng }}
              </slot>
            </li>
          </ul>
        </vue-scroller>

        <!--
          @slot Use this slot to display information after the list.
            @binding {Array} items         The full list of items.
            @binding {Array} filteredItems The filtered list of items.
        -->
        <slot
          name="after-list"
          :items="items"
          :filtered-items="filteredItems"
          :selected-item="selectedItem"
          :filter-features-in-view="filterFeaturesInView" />
      </template>
    </div>
    <div :class="(classes.region || {}).panel || {}">
      <transition v-bind="transitions.panel || {}">
        <div v-if="selectedItem" :key="selectedItem.id" :class="classes.panel || {}">
          <!--
            @slot Use this slot to display content inside the panel.
              @binding {Object}   item  The selected item.
              @binging {Function} close A function to close the panel
          -->
          <slot name="panel" :item="selectedItem" :close="() => (selectedItem = null)">
            <div>{{ selectedItem }}</div>
          </slot>
        </div>
      </transition>
    </div>
  </div>
</template>

<script>
  import MapboxCluster from '../MapboxCluster';
  import MapboxGeocoder from '../MapboxGeocoder';
  import MapboxMap from '../MapboxMap';
  import VueScroller from './VueScroller';

  export default {
    components: {
      MapboxCluster,
      MapboxGeocoder,
      MapboxMap,
      VueScroller,
    },
    props: {
      /**
       * A list of items to display.
       * The only required properties are `lat` and `lng` and `id`.
       *
       * @type {Array<{ lat: Number, lng: number, id: string, [prop: string]: unknown }>}
       */
      items: {
        type: Array,
        required: true,
      },

      /**
       * The zoom level to use when zooming in on an item.
       *
       * @type {Number}
       */
      itemZoomLevel: {
        type: Number,
        default: 14,
      },
      /**
       * A Mapbox access token.
       * @type {Object}
       */
      accessToken: {
        type: String,
        default: 'no-token',
      },
      /**
       * Props for the MapboxMap component.
       *
       * @see  https://vue-mapbox-gl.meta.fr/components/MapboxMap.html#props
       * @type {Object}
       */
      mapboxMap: {
        type: Object,
        default: () => ({}),
      },
      /**
       * Props fof the MapboxCluster component.
       *
       * @see  https://vue-mapbox-gl.meta.fr/components/MapboxCluster.html#props
       * @type {Object}
       */
      mapboxCluster: {
        type: Object,
        default: () => ({}),
      },
      /**
       * Props for the MapboxGeocoder component.
       *
       * @see  https://vue-mapbox-gl.meta.fr/components/MapboxGeocoder.html#props
       * @type {Object}
       */
      mapboxGeocoder: {
        type: Object,
        default: () => ({}),
      },

      /**
       * Configuration for each transition component.
       * @type {Object}
       */
      transitions: {
        type: Object,
        default: () => ({
          loader: {
            map: {},
            search: {},
            list: {},
            default: {},
          },
          panel: {},
        }),
      },

      /**
       * Define custom classes for each element of the component.
       * @type {Object}
       */
      classes: {
        type: Object,
        default: () => {
          const root = 'store-locator';
          const bem = (name, modifier = '') => `${root}__${name}${modifier ? `--${modifier}` : ''}`;
          return {
            root,
            region: {
              map: [ bem('region'), bem('region', 'map') ],
              search: [ bem('region'), bem('region', 'search') ],
              list: [ bem('region'), bem('region', 'list') ],
              panel: [ bem('region'), bem('region', 'panel') ],
            },
            map: bem('map'),
            search: bem('search'),
            list: bem('list'),
            listItem: bem('list-item'),
            panel: bem('panel'),
          };
        },
      },
    },
    data() {
      return {
        isLoading: true,
        map: null,
        mapIsMoving: false,
        selectedItem: null,
        filteredItems: this.items.map((item) => item),
        listIsLoading: false,
      };
    },
    computed: {
      geoJson() {
        return {
          type: 'FeatureCollection',
          features: this.items.map(this.itemToGeoJsonFeature),
        };
      },
      filteredGeoJson() {
        return {
          type: 'FeatureCollection',
          features: this.filteredItems.map(this.itemToGeoJsonFeature),
        };
      },
    },
    methods: {
      /**
       * Transform an item into a valid GeoJSON feature.
       *
       * @param  {Object} item The item to format.
       * @return {Feature}     A GeoJSON feature.
       */
      itemToGeoJsonFeature({ lat, lng, ...properties }) {
        return {
          type: 'Feature',
          geometry: {
            type: 'Point',
            coordinates: [ lng, lat ],
          },
          properties,
        };
      },

      /**
       * Transform a GeoJSON feature into an item.
       *
       * @param  {Geometry} options.geometry   A GeoJSON geometry object.
       * @param  {Object}   options.properties The feature properties.
       * @return {Object}                      An item.
       */
      geoJsonFeatureToItem({ geometry, properties }) {
        return {
          lat: geometry.coordinates[1],
          lng: geometry.coordinates[0],
          ...properties,
        };
      },

      /**
       * Handler for the geocoder result event.
       *
       * @param {Object} result The place selected in the geocoder component.
       */
      onGeocoderResult({ result }) {
        if (result.bbox) {
          this.map.fitBounds(result.bbox);
        } else if (result.center) {
          this.map.flyTo({ center: result.center });
        }
      },

      /**
       * Propagate the `mb-created` event from the MapboxGeocoder component.
       *
       * @param  {Geocoder} geocoder The geocoder instance.
       * @return {void}
       */
      onGeocoderCreated(geocoder) {
        this.$emit('geocoder-created', geocoder);
      },

      /**
       * Handler for the map created event.
       *
       * @param  {MapboxMap} instance The Mapbox instance.
       */
      async onMapCreated(instance) {
        this.map = instance;
        this.$emit('map-created', instance);
        this.filterFeaturesInView();
      },

      /**
       * Handler for the map load event.
       */
      async onMapLoad() {
        await this.$nextTick();
        this.isLoading = false;

        if (this.$listeners.load) {
          console.warn(
            '[StoreLocator]',
            'The `@load` event is deprecated, replace it with the `@map-load` event instead.',
          );
        }

        this.$emit('load', this.map);
        this.$emit('map-load', this.map);
      },

      /**
       * Handler for the map's movestart event.
       */
      onMapMovestart() {
        this.mapIsMoving = true;
        this.$nextTick(() => {
          this.listIsLoading = true;
        });
      },

      /**
       * Handler for the map's moveend event.
       */
      onMapMoveend() {
        this.mapIsMoving = false;
        this.filterFeaturesInView();
      },

      /**
       * Filter the features in view.
       */
      async filterFeaturesInView() {
        this.listIsLoading = true;
        const mapBounds = this.map.getBounds();
        const center = this.map.getCenter();

        this.filteredItems = this.items
          .filter(({ lng, lat }) => mapBounds.contains([ lng, lat ]))
          .sort((a, b) => {
            const distanceFromA = center.distanceTo(a);
            const distanceFromB = center.distanceTo(b);

            if (distanceFromA < distanceFromB) {
              return -1;
            }

            if (distanceFromA > distanceFromB) {
              return 1;
            }

            return 0;
          });
        await this.$nextTick();
        this.listIsLoading = false;
      },

      /**
       * Handler for the click event on a list item.
       * @param {Feature} store A GeoJSON feature.;
       */
      onListItemClick(item) {
        this.selectedItem = item;
        this.$emit('select-item', item);

        const { lat, lng } = this.map.getCenter();

        // Do not trigger flyTo if the map is almost already centered
        if (Math.abs(lng - item.lng) > 0.0001 && Math.abs(lat - item.lat) > 0.0001) {
          this.map.flyTo({ center: [ item.lng, item.lat ], zoom: this.itemZoomLevel });
        }
      },

      /**
       * Handler for the click event on a GeoJSON feature.
       *
       * @param  {Feature} feature The GeoJSON feature being clicked.
       */
      onClusterFeatureClick(feature, event) {
        const item = this.items.find(({ id }) => id === feature.properties.id);
        this.$emit('cluster-feature-click', feature, event);

        if (item) {
          this.$emit('select-item', item);
          this.selectedItem = item;
          this.map.flyTo({ center: feature.geometry.coordinates, zoom: this.itemZoomLevel });
        }
      },
      onClusterFeatureMouseenter(...args) {
        this.$emit('cluster-feature-mouseenter', ...args);
      },
      onClusterFeatureMouseleave(...args) {
        this.$emit('cluster-feature-mouseleave', ...args);
      },
      onClusterClusterClick(...args) {
        this.$emit('cluster-cluster-click', ...args);
      },
    },
  };
</script>
