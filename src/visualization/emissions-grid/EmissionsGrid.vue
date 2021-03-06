<template lang="pug">
#container
  .map-container
    #mymap

  .status-blob(v-if="loadingText"): p {{ loadingText }}

  left-data-panel.left-panel(v-if="!loadingText" title="Emissions Grid")
    .dashboard-panel
      .pollution-settings
        h4.heading Pollutant
        .pollutants
          .hey(v-for="p in pollutants")
            button.button.pollutant(
              :class="{'is-warning': p===pollutant, 'is-gray': p!==pollutant}"
              @click="clickedPollutant(p)") {{p}}
        h4.heading Time of Day
        .slider-box
          time-slider.time-slider(:bind="currentTime"
                                  :initialTime="currentTime"
                                  @change="changedSlider")
      .theme-choices
        img.theme-button(v-for="theme in themes"
                        :class="{'selected-theme': theme.name === chosenTheme.name}"
                        :src="theme.icon"
                        @click="clickedTheme(theme)")
</template>

<script lang="ts">
'use strict'
import * as timeConvert from 'convert-seconds'
import mapboxgl from 'mapbox-gl'
import nprogress from 'nprogress'
import { LngLat } from 'mapbox-gl/dist/mapbox-gl'
import pako from 'pako'
import readBlob from 'read-blob'

import AuthenticationStore from '@/auth/AuthenticationStore'
import Coords from '@/components/Coords'
import Config from '@/config/Config'
import EventBus from '@/EventBus.vue'
import FileAPI from '@/communication/FileAPI'
import ProjectStore from '@/project/ProjectStore'
import LeftDataPanel from '@/components/LeftDataPanel.vue'
import sharedStore from '@/SharedStore'
import TimeSlider from '@/components/TimeSlider.vue'
import { Vue, Component, Prop, Watch } from 'vue-property-decorator'
import { inferno, viridis } from 'scale-color-perceptual'
import MyWorker from './MyWorker'

interface MapElement {
  lngLat: LngLat
  features: any[]
}

interface NOXEntry {
  id: string
  time: number
  nox: number
}

interface Point {
  id: string
  lon: number
  lat: number
  events: any[]
}

@Component({
  components: {
    TimeSlider,
    LeftDataPanel,
  },
})
class EmissionsGrid extends Vue {
  @Prop({ type: String, required: true })
  private projectId!: string

  @Prop({ type: String, required: true })
  private vizId!: string

  @Prop({ type: ProjectStore, required: true })
  private projectStore!: ProjectStore

  @Prop({ type: AuthenticationStore, required: true })
  private authStore!: AuthenticationStore

  @Prop({ type: FileAPI, required: true })
  private fileApi!: FileAPI

  private currentTime: number = 0
  private firstEventTime: number = 0
  private firstLoad: boolean = true
  private mymap!: mapboxgl.Map
  private mapExtentXYXY: any = [180, 90, -180, -90]
  private initialMapExtent: any = null
  private noxLocations: any
  private sharedState: any = sharedStore.state

  private loadingText: string = 'Emissions Grid'
  private visualization: any = null
  private project: any = {}
  private projection!: string

  private pollutant: string = ''
  private pollutantsMaxValue: { [id: string]: number } = {}
  private pollutants: any = []
  private dataLookup: any = {}

  private timeBins: any = []
  private selectedTimeBin = 0

  private _myWorker: any

  private themes: any = [
    {
      name: 'Inferno',
      colorRamp: 'revInferno',
      icon: '/infernwhite.png',
      style: 'mapbox://styles/mapbox/light-v9',
    },
    {
      name: 'Viridis',
      colorRamp: 'revViridis',
      icon: '/viriwhite.png',
      style: 'mapbox://styles/mapbox/light-v9',
    },
    {
      name: 'Indarko',
      colorRamp: 'colorInferno',
      icon: '/inferno.png',
      style: 'mapbox://styles/mapbox/dark-v9',
    },
    {
      name: 'Viridark',
      colorRamp: 'colorViridis',
      icon: '/viridis.png',
      style: 'mapbox://styles/mapbox/dark-v9',
    },
  ]

  // choose your colormap: for emissions we'll use inferno
  // https://www.npmjs.com/package/scale-color-perceptual
  private chosenTheme: any = this.themes[0]

  public beforeDestroy() {
    if (this._myWorker) this._myWorker.destroy()
  }

  public created() {
    sharedStore.setFullPage(true)
  }

  public destroyed() {
    sharedStore.setFullPage(false)
  }
  public async fetchEmissionsBins(): Promise<any> {
    const result = await fetch(`${Config.emissionsServer}/${this.vizId}/startTimes`, {
      mode: 'cors',
      headers: { Authorization: 'Bearer ' + this.authStore.state.accessToken },
    })

    if (result.ok) {
      try {
        const json = await result.json()
        console.log({ json })
        return json
      } catch (e) {
        throw new Error(e)
      }
    } else if (result.status === 401) {
      throw new Error('Unauthorized: ' + (await result.text()))
    } else {
      throw new Error(await result.text())
    }
  }

  // VUE LIFECYCLE: mounted
  public async mounted() {
    this.visualization = await this.fileApi.fetchVisualization(this.projectId, this.vizId)
    this.project = await this.fileApi.fetchProject(this.projectId)

    sharedStore.setBreadCrumbs([
      { label: this.visualization.title, url: '/' },
      { label: this.visualization.project.name, url: '/' },
    ])

    if (this.visualization.parameters.Projection) this.projection = this.visualization.parameters.Projection.value

    // do things that can only be done after MapBox is fully initialized
    this.mymap = new mapboxgl.Map({
      bearing: 0,
      // center: [x,y], // lnglat, not latlng (think of it as: x,y)
      container: 'mymap',
      logoPosition: 'top-left',
      style: this.chosenTheme.style,
      pitch: 0,
      zoom: 14,
    })

    try {
      this.initialMapExtent = localStorage.getItem(this.vizId + '-bounds')
      if (this.initialMapExtent) {
        const lnglat = JSON.parse(this.initialMapExtent)

        const mFac = this.isMobile ? 0 : 1
        const padding = { top: 50 * mFac, bottom: 100 * mFac, right: 100 * mFac, left: 300 * mFac }

        this.mymap.fitBounds(lnglat, {
          padding,
          animate: false,
        })
      }
    } catch (e) {
      console.log(e)
    }
    this.mymap.on('style.load', this.mapIsReady)
  }

  private isMobile() {
    const w = window
    const d = document
    const e = d.documentElement
    const g = d.getElementsByTagName('body')[0]
    const x = w.innerWidth || e.clientWidth || g.clientWidth
    const y = w.innerHeight || e.clientHeight || g.clientHeight

    return x < 640
  }

  private clickedTheme(theme: any) {
    console.log('changing theme: ' + theme)

    if (theme.style !== this.chosenTheme.style) {
      console.log('changing style: ' + theme.style)
      this.mymap.setStyle(theme.style)
    }

    this.chosenTheme = theme
    this.mymap.setPaintProperty('hex-layer', 'fill-color', ['get', theme.colorRamp]) // fill-extrusion-color
  }

  private setJsonSource() {
    const p = this.pollutant ? this.pollutant : this.pollutants[0]
    const storageKey = p + ':' + this.timeBins[this.selectedTimeBin]

    const jsonData = this.dataLookup[storageKey]
    try {
      this.mymap.addSource('hexagons', {
        data: jsonData,
        type: 'geojson',
      })
    } catch (e) {
      console.log(e)
    }
  }

  private addJsonToMap() {
    this.mymap.addLayer(
      {
        id: 'hex-layer',
        source: 'hexagons',
        type: 'fill',
        paint: {
          'fill-color': ['get', this.chosenTheme.colorRamp],
          'fill-opacity': ['get', 'op'],
        },
      },
      'road-street'
      // 'water', 'tunnel-street-low' // water, road-street, road-secondary-tertiary, building...
    ) // layer gets added just *under* this MapBox-defined layer.
  }

  private setMapExtent() {
    localStorage.setItem(this.vizId + '-bounds', JSON.stringify(this.mapExtentXYXY))
    this.mymap.fitBounds(this.mapExtentXYXY, {
      padding: { top: 50, bottom: 100, right: 100, left: 300 },
      animate: false,
    })
  }

  private clickedPollutant(p: string) {
    console.log(p)
    this.pollutant = p

    const storageKey = p + ':' + this.timeBins[this.selectedTimeBin]
    const jsonData = this.dataLookup[storageKey]

    if (jsonData) (this.mymap.getSource('hexagons') as any).setData(jsonData)
  }

  private changedSlider(seconds: number) {
    this.currentTime = seconds

    let bin = -1
    for (const startTime of this.timeBins) {
      if (startTime > seconds) break
      bin++
    }

    this.selectedTimeBin = bin
    const storageKey = this.pollutant + ':' + this.timeBins[bin]

    const jsonData = this.dataLookup[storageKey]
    if (jsonData) (this.mymap.getSource('hexagons') as any).setData(jsonData)
  }

  private async loadData() {
    const bins = await this.fetchEmissionsBins()
    const sortedBins = bins.sort((a: number, b: number) => a - b)

    // spawn transit helper web worker
    this._myWorker = await MyWorker.create({
      accessToken: this.authStore.state.accessToken,
      bins: sortedBins,
      cellSize: this.visualization.parameters['Cell size'].value,
      projectId: this.projectId,
      projection: this.projection,
      url: `${Config.emissionsServer}/${this.vizId}/bin?startTime=`,
    })

    this.loadingText = 'Loading Emissions Grid Data...'
    const data = await this._myWorker.loadData()

    this.dataLookup = data.dataLookup
    this.pollutantsMaxValue = data.pollutantsMaxValue
    this.mapExtentXYXY = data.mapExtentXYXY
    this.pollutants = data.pollutants
    this.timeBins = data.timeBins

    this.setMapExtent()
    this.pollutant = this.pollutants[0]
  }

  // this is called every time map setStyle is called, too
  private async mapIsReady() {
    if (this.firstLoad) {
      this.firstLoad = false
      this.mymap.addControl(new mapboxgl.NavigationControl(), 'top-right')

      await this.loadData()

      if (!this.initialMapExtent) {
        this.mymap.jumpTo({ center: [this.mapExtentXYXY[0], this.mapExtentXYXY[1]], zoom: 13 })
        this.mymap.fitBounds(this.mapExtentXYXY, { padding: 150 })
      }
    }
    this.setJsonSource()
    this.addJsonToMap()

    this.loadingText = ''
    nprogress.done()
  }
}

sharedStore.addVisualizationType({
  component: EmissionsGrid,
  typeName: 'emissions',
  prettyName: 'Emissions Grid',
  description: 'Show emissions at gridpoints',
  requiredFileKeys: ['Events', 'Network'],
  requiredParamKeys: ['Projection', 'Cell size', 'Smoothing radius', 'Time bin size'],
})

export default EmissionsGrid
</script>

<style scoped>
#container {
  display: grid;
  grid-template-columns: auto 1fr auto;
  grid-template-rows: 1fr auto;
  width: 100%;
}

.map-container {
  background-color: #eee;
  grid-column: 1 / 4;
  grid-row: 1 / 3;
  display: flex;
  flex-direction: column;
}

#mymap {
  height: 100%;
  width: 100%;
}

.loading-message {
  grid-row: 1 / 2;
  grid-column: 1 / 4;
  overflow: hidden;
  opacity: 0.8;
}

.slider-box {
  grid-row: 2 / 3;
  grid-column: 1 / 4;
  background-color: white;
  z-index: 2;
  border: solid 1px;
  border-color: #ccc;
  border-radius: 4px;
  box-shadow: 0px 1px 3px rgba(0, 0, 0, 0.2);
  margin-top: 2rem;
}

h2,
h3 {
  color: #6666ff;
  margin-top: 15px;
  margin-bottom: 3px;
}

.lead {
  text-align: center;
  color: #555;
  font-family: 'Oswald', sans-serif;
}

/* `:focus` is linked to `:hover` for basic accessibility */
a:hover,
a:focus {
  text-decoration: none;
}

.status-blob {
  background-color: #fff;
  box-shadow: 0 0 8px #00000040;
  opacity: 0.9;
  margin: auto 0px auto -10px;
  padding: 3rem 0px;
  text-align: center;
  grid-column: 1 / 3;
  grid-row: 1 / 3;
  z-index: 8;
  border-top: solid 1px #479ccc;
  border-bottom: solid 1px #479ccc;
}

.status-blob p {
  color: #0066a2; /* #ffa; /* #0066a1; */
}

.left-panel {
  grid-column: 1 / 2;
  grid-row: 1 / 3;
  display: flex;
  flex-direction: column;
  width: 16rem;
  overflow-y: auto;
}

.info-header {
  background-color: #097c43;
  padding: 0.5rem 0rem;
  border-top: solid 1px #888;
  border-bottom: solid 1px #888;
  text-align: center;
}

.pollution-settings {
  margin: 0rem 0.5rem 0.5rem 0.5rem;
  padding: 0rem 0.25rem;
  flex-grow: 1;
}

.pollutant {
  width: 100%;
  margin-bottom: 0.25rem;
}

.heading {
  text-align: left;
  color: black;
  margin-top: 2rem;
}

.theme-choices {
  display: flex;
  flex-direction: row;
  margin: 2px auto;
}

.theme-button {
  width: 3rem;
  height: 3rem;
  margin: 0rem 0.25rem 0.5rem 0.25rem;
  padding: 1px 1px;
  background-color: black;
}

.theme-button:hover {
  background-color: cyan;
}

.selected-theme {
  background-color: #6f6;
}

.dashboard-panel {
  display: flex;
  flex-direction: column;
  flex-grow: 1;
}

.selected-theme:hover {
  background-color: #6f6;
}
</style>
