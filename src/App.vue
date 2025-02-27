<template lang="pug">

#app
  #top
    SelectSource(@select="updateSource")

  #middle
    #sidepanel
      h3 Params
      input(id="showBoundaries" type="checkbox" v-model="showBoundaries")
      label(for="showBoundaries") Show tile boundaries
      br
      input(id="fixedZoom" type="checkbox", v-model="fixedZoom")
      label(for="fixedZoom") Fix zoom to 
      code {{ fixedZoom ? fixedZoomValue : zoom }}
      br
      label(v-if="fixedZoom && fixedZoomValue > zoom") Warning: mapbox-gl can't underzoom.
      h3 Info
      InfoMapBox(:zoom="zoom")
      h3 Layers
      #layers
        AttributesTable(:layers="layers" :values="values" @clickAttribute="summariseAttribute" :focusedAttribute="focusedAttribute" :focusedLayer="focusedLayer")
        //- AttributeSummary(:sortedValueCounts="sortedValueCounts" :attributeName="focusedAttribute" @hoverValue="value => { focusedValue = value}" :focusedValue="focusedValue")
    MapComponent(:layers="layers" :xyzUrl="xyzUrl" :tms="tms" @values-update="updateValues" @move="updateAttributeSummary" @zoom="updateZoom" :focusValue="focusedValue" :focusLayer="focusedLayer" :focusField="focusedAttribute")
  #bottom
    .credit Forked from <a href="https://github.com/stevage/vector-inspector">github.com/stevage/vector-inspector</a> - Steve Bennett (@stevage1) 
</template>

<script>
require('mapbox-gl-inspect/dist/mapbox-gl-inspect.css');
import AttributesTable from './components/AttributesTable';
import InfoMapBox from './components/InfoMapBox';
import SelectSource from './components/SelectSource';
import AttributeSummary from '@/components/AttributeSummary.vue';
import MapComponent from '@/components/Map.vue';

const request = require('request');
const VectorTile = require('@mapbox/vector-tile').VectorTile;
const Pbf = require('pbf');
const zlib = require('zlib');
const cssColors = [
    '#1f78b4',
    // '#b2df8a',
    '#33a02c',
    '#e31a1c',
    '#ff7f00',
    'hsl(90,50%,30%)', // '#a6cee3',
    '#fb9a99',
    '#cab2d6',
    '#fdbf6f',
    'hsl(240,90%,60%)', // '#a6cee3',
    '#6a3d9a',
    '#bb5',
    '#b15928',
    ...Object.keys(require('css-color-names')),
];

//http://localhost:4040/buildings/13/2411/3079.pbf
//https://tile.nextzen.org/tilezen/vector/v1/512/all/14/4826/6157.mvt?api_key=

export default {
    name: 'app',
    components: {
        AttributesTable,
        AttributeSummary,
        InfoMapBox,
        SelectSource,
        MapComponent,
    },
    data: () => ({
        url: "",
        layers: [],
        values: {},
        cors: false,
        tms: false,
        zyx: !!window.location.hash.match(/zyx/),
        tilejsonXyzUrl: '',
        showBoundaries: false,
        focusedLayer: undefined,
        focusedAttribute: undefined,
        focusedValue: undefined,
        valueCounts: {},
        sortedValueCounts: [],
        zoom: undefined,
        fixedZoom: undefined,
        fixedZoomValue: undefined,
    }),
    mounted() {
        window.setTimeout(() => {
            if (localStorage.getItem('url') && !this.url) {
                // this.url = localStorage.getItem("url");
            }
        }, 1000);

        window.App = this;
    },
    computed: {
        xyzUrl() {
            if (this.tilejsonXyzUrl) {
                return this.tilejsonXyzUrl;
            }
            return this.url.replace(
                /([/=])\d+\/\d+\/\d+/,
                this.zyx ? '$1{z}/{y}/{x}' : '$1{z}/{x}/{y}',
            );
        },
        tileJsonUrl() {
            return this.url.replace(
                /\/\d+\/\d+\/\d+(\.[a-zA-Z]*)?/,
                '/index.json',
            );
        },
        urlParams() {
            return this.url + this.zyx;
        },
    },
    watch: {
        urlParams: {
            handler() {
                this.refresh();
                localStorage.setItem('url', this.url);
                const loc = ((window.location.hash || '').match(
                    /&loc=[.0-9/-]+/,
                ) || [])[0];
                window.location.hash =
                    `?url=${this.url}` +
                    (this.zyx ? '&zyx' : '') +
                    (loc ? loc : '');
                // window.location.hash = window.location.hash.replace(
                //     /(\??&?url=[^&]+|$)/,
                //     `&url=${this.url}`,
                // );
                // if (window.location.hash.m
            },
            immediate: true,
        },
        cors() {
            this.refresh();
        },
        showBoundaries() {
            window.map.showTileBoundaries = this.showBoundaries;
        },
        fixedZoom(isFixed) {
            const source = window.map.getSource('source');
            if (isFixed) {
                this.fixedZoomValue = this.zoom;
                source.maxzoom = this.zoom;
                source.minzoom = this.zoom;
            } else {
                source.maxzoom = 9;
                source.minzoom = 0;
            }
        },
    },
    methods: {
        summariseAttribute(attribute, layer) {
            this.focusedAttribute = attribute;
            this.focusedLayer = layer;
            this.updateAttributeSummary();
        },
        updateAttributeSummary() {
            if (!this.focusedAttribute) {
                return;
            }
            const features = window.map.querySourceFeatures('source', {
                sourceLayer: this.focusedLayer,
            });
            // we have to use Map instead of a regular object in order to preserve the type of numeric keys
            this.valueCounts = new Map();
            for (const {
                properties,
                geometry: { type },
            } of features) {
                const value = properties[this.focusedAttribute];
                const counts = this.valueCounts.get(value) || [0, 0, 0, 0, 0]; // total, point, line, polygon, multipolygon
                const countFieldIndex =
                    ['Point', 'LineString', 'Polygon', 'MultiPolygon'].indexOf(type) + 1;
                counts[0] += 1;
                counts[countFieldIndex] += 1;

                this.valueCounts.set(value, counts);
            }
            this.sortedValueCounts = Array.from(
                this.valueCounts.entries()
            ).sort(([a], [b]) => a > b ? 1 : -1);
        },
        updateZoom() {
            this.zoom = Math.floor(window.map.getZoom());
        },
        updateSource(source) {
            this.url = "http://localhost:8080"+source+"0/0/0.pbf";
            this.fixedZoom = false;
        },
        updateValues(newValues) {
            this.values = newValues;
        },
        refresh() {
            const req = (url, params, cb) => {
                return request(
                    {
                        url: this.cors
                            ? 'https://cors-anywhere.herokuapp.com/' + url
                            : url,
                        headers: this.cors
                            ? { Origin: window.location.host }
                            : {},
                        ...params,
                    },
                    cb,
                );
            };
            if (this.url.match(/\d+\/\d+\/\d+/)) {
                req(this.url, { encoding: null }, (err, response, body) => {
                    try {
                        body = zlib.gunzipSync(body);
                    } catch (e) {
                        // probably not a zip file
                        // console.error(e);
                    }
                    if (!body) {
                        this.layers = [];
                        console.log('No body when requesting ', this.url);
                        return;
                    }
                    const tile = new VectorTile(new Pbf(body));
                    console.log(tile);
                    // console.log('Vector source layers found: ');
                    this.layers = Object.keys(tile.layers).map((id, i) => ({
                        ...tile.layers[id],
                        _color: cssColors[i],
                    }));
                });

                req(this.tileJsonUrl, { json: true }, (err, response, body) => {
                    if (body.center) {
                        window.map.panTo(body.center, { zoom: body.center[2] });
                    }
                    // console.log(body);
                });
            } else if (this.url.match(/\.json/)) {
                req(this.url, { json: true }, (err, response, body) => {
                    this.layers = body.vector_layers.map((layer, i) => ({
                        _color: cssColors[i],
                        name: layer.id,
                        _keys: Object.keys(layer.fields),
                    }));
                    this.tilejsonXyzUrl = body.tiles[0];
                });
            } else {
                this.layers = [];
                this.values = {};
            }
        },
    },
};
</script>

<style>
/* @import '../node_modules/mapbox-gl-inspect/dist/mapbox-gl-inspect.css'; */
body {
    margin: 0;
    padding: 0;
}
#app {
    font-family: 'Avenir', Helvetica, Arial, sans-serif;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
    color: #2c3e50;
    display: flex;
    flex-direction: column;
    height: 100vh;
    width: 100vw;
    overflow: hidden;
}

#top {
    flex-grow: 0;
    padding: 1em;
}

#middle {
    flex-grow: 1;
    display: flex;
    min-height: 0;
}

#sidepanel {
    width: 300px;
    flex-grow: 0;
    background: white;
    border: 1px solid lightgrey;
    padding: 0 1em;
    overflow-y: scroll;
}

#layers {
    /* display:none; */
}

#url {
    width: 100%;
}

.credit {
    font-size: 10pt;
    color: grey;
}

.mapboxgl-popup {
    background: white;
    /* max-width:20em; */
    display: inline-block;
}

.mapbox-gl-inspect_popup {
    font-size: 10px;
    pointer-events: none;
}

.mapbox-gl-inspect_property-value {
}

.mapboxgl-ctrl-inspect {
    /* height:50px; */
    display: none;
}

.mapboxgl-popup,
.mapboxgl-popup * {
    pointer-events: none;
}

.mapboxgl-popup {
    max-height: 500px;
    overflow-y: hidden;
}

.fixed {
    font-family: Consolas, Monaco, 'Courier New', Courier, monospace;
}
</style>
