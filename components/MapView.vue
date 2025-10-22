<template>
  <div ref="mapEl" class="map"></div>

  <!-- Tooltip -->
  <div v-if="hover" class="tooltip" :style="{ left: hover.x + 'px', top: hover.y + 'px' }">
    <div class="name">{{ hover.props.name }}</div>
    <div class="row"><span>Тип:</span> <b>{{ hover.props.isForte ? 'Forte' : hover.props.bank || 'Конкурент' }}</b></div>
    <div class="row" v-if="hover.props.avgCashBalanceKZT"><span>Остаток:</span> <b>{{ fmtKZT(hover.props.avgCashBalanceKZT) }}</b></div>
    <div class="row" v-if="hover.props.withdrawalFreqPerDay"><span>Снятия/день:</span> <b>{{ hover.props.withdrawalFreqPerDay }}</b></div>
  </div>
</template>

<script setup lang="ts">
import { onMounted, onBeforeUnmount, ref, watch, computed } from 'vue'
import maplibregl from 'maplibre-gl'
import 'maplibre-gl/dist/maplibre-gl.css'

type ATM = {
  id:number; name:string; lat:number; lng:number; isForte:boolean;
  avgCashBalanceKZT?:number; withdrawalFreqPerDay?:number; bank?:string; district?:string;
}
type Flow = { fromId:number; toId:number; weight:number }     // 0..1
type HeatPoint = { lat:number; lng:number; weight:number }    // 0..1

const props = defineProps<{
  forteAtms: ATM[]
  competitorAtms: ATM[]
  flows: Flow[]
  heatPoints: HeatPoint[]
  showHeatmap: boolean
}>()

const mapEl = ref<HTMLDivElement | null>(null)
let map: maplibregl.Map | null = null
const hover = ref<{ x:number, y:number, props:any } | null>(null)

const OSM_STYLE = {
  version: 8,
  sources: {
    osm: {
      type: 'raster',
      tiles: ['https://tile.openstreetmap.org/{z}/{x}/{y}.png'],
      tileSize: 256,
      attribution: '© <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
    }
  },
  layers: [{ id:'osm', type:'raster', source:'osm' }]
}

/* ---------- helpers ---------- */

// «размножаем» каждую точку heatmap в кольца с убывающим весом — получаем гладкие «лужи»
function densify(pts: HeatPoint[]): HeatPoint[] {
  if (!pts.length) return []
  const lat0 = pts[0].lat
  const m2degLat = 1 / 111_320
  const m2degLon = 1 / (111_320 * Math.cos(lat0 * Math.PI/180))

  const out: HeatPoint[] = []
  for (const p of pts) {
    out.push(p) // базовая точка
    const radii = [60, 120, 200] // метры
    const directions = 24
    for (let r=0; r<radii.length; r++) {
      const R = radii[r]
      for (let k=0; k<directions; k++) {
        const ang = (2*Math.PI*k)/directions
        const lat = p.lat + (R * Math.sin(ang)) * m2degLat
        const lng = p.lng + (R * Math.cos(ang)) * m2degLon
        const decay = Math.exp(-Math.pow((r+1)/3, 2)) // плавное затухание
        out.push({ lat, lng, weight: Math.max(0, Math.min(1, p.weight * decay)) })
      }
    }
  }
  return out
}
function fmtKZT(n:number){
  return new Intl.NumberFormat('ru-KZ',{style:'currency',currency:'KZT',maximumFractionDigits:0}).format(n)
}

/* ---------- feature collections ---------- */
const forteFC = computed(() => ({
  type:'FeatureCollection',
  features: props.forteAtms.map(a => ({
    type:'Feature', properties:a, geometry:{ type:'Point', coordinates:[a.lng, a.lat] }
  }))
}))
const competitorsFC = computed(() => ({
  type:'FeatureCollection',
  features: props.competitorAtms.map(a => ({
    type:'Feature', properties:a, geometry:{ type:'Point', coordinates:[a.lng, a.lat] }
  }))
}))
const flowsFC = computed(() => ({
  type:'FeatureCollection',
  features: props.flows.map(f => {
    const from = props.forteAtms.find(a => a.id===f.fromId)
    const to   = props.forteAtms.find(a => a.id===f.toId)
    if (!from || !to) return null
    return {
      type:'Feature',
      properties:{ weight:f.weight },
      geometry:{ type:'LineString', coordinates:[[from.lng, from.lat],[to.lng, to.lat]] }
    }
  }).filter(Boolean) as any[]
}))
const heatFC = computed(() => {
  const dense = densify(props.heatPoints)
  return {
    type:'FeatureCollection',
    features: dense.map(p => ({
      type:'Feature',
      properties:{ weight:p.weight },
      geometry:{ type:'Point', coordinates:[p.lng, p.lat] }
    }))
  }
})

function updateSources(){
  if(!map) return
  ;(map.getSource('forte') as maplibregl.GeoJSONSource)?.setData(forteFC.value)
  ;(map.getSource('competitors') as maplibregl.GeoJSONSource)?.setData(competitorsFC.value)
  ;(map.getSource('flows') as maplibregl.GeoJSONSource)?.setData(flowsFC.value)
  ;(map.getSource('heat') as maplibregl.GeoJSONSource)?.setData(heatFC.value)

  const all = [...props.forteAtms, ...props.competitorAtms]
  if (all.length) {
    const lons = all.map(p=>p.lng), lats = all.map(p=>p.lat)
    map!.fitBounds([[Math.min(...lons), Math.min(...lats)], [Math.max(...lons), Math.max(...lats)]], { padding:40, duration:0 })
  }
}

function applyVisibility(){
  if(!map) return
  const showHeat = props.showHeatmap ? 'visible' : 'none'
  const showFlows = props.showHeatmap ? 'none' : 'visible'
  if (map.getLayer('heatmap')) map.setLayoutProperty('heatmap', 'visibility', showHeat)
  if (map.getLayer('heat-dots')) map.setLayoutProperty('heat-dots', 'visibility', showHeat)
  if (map.getLayer('flow-lines')) map.setLayoutProperty('flow-lines', 'visibility', showFlows)
}

/* ---------- lifecycle ---------- */
onMounted(() => {
  map = new maplibregl.Map({
    container: mapEl.value as HTMLDivElement,
    style: OSM_STYLE,
    center: [71.43, 51.16],
    zoom: 11,
    attributionControl: true
  })
  map.addControl(new maplibregl.NavigationControl({ showCompass:false }), 'top-right')
  map.addControl(new maplibregl.ScaleControl({ unit:'metric' }), 'bottom-left')

  map.on('load', () => {
    // точки
    map!.addSource('forte', { type:'geojson', data: forteFC.value })
    map!.addLayer({
      id:'forte-points', type:'circle', source:'forte',
      paint:{
        'circle-radius':6, 'circle-color':'#00B8D9',
        'circle-stroke-width':1, 'circle-stroke-color':'#0EA5E9'
      }
    })

    map!.addSource('competitors', { type:'geojson', data: competitorsFC.value })
    map!.addLayer({
      id:'competitor-points', type:'circle', source:'competitors',
      paint:{
        'circle-radius':5, 'circle-color':'#EF4444',
        'circle-stroke-width':1, 'circle-stroke-color':'#F87171'
      }
    })

    // маршруты
    map!.addSource('flows', { type:'geojson', data: flowsFC.value })
    map!.addLayer({
      id:'flow-lines', type:'line', source:'flows',
      paint:{
        'line-width':['interpolate',['linear'],['get','weight'],0,1,1,6],
        'line-color':'#7C3AED', 'line-opacity':0.35
      }
    })

    // HEATMAP (радужный «жирный» градиент)
    map!.addSource('heat', { type:'geojson', data: heatFC.value })
    map!.addLayer({
      id:'heatmap', type:'heatmap', source:'heat',
      paint:{
        'heatmap-weight': ['interpolate',['linear'],['get','weight'], 0,0, 0.2,0.2, 0.5,0.6, 1,1],
        'heatmap-radius': ['interpolate',['linear'],['zoom'], 10,20, 12,30, 14,40, 16,55],
        'heatmap-intensity': ['interpolate',['linear'],['zoom'], 10,1.2, 14,2.2],
        'heatmap-color': [
          'interpolate', ['linear'], ['heatmap-density'],
          0.00, 'rgba(0,0,0,0)',
          0.10, '#0000ff',
          0.30, '#00ffff',
          0.50, '#00ff00',
          0.70, '#ffff00',
          0.85, '#ff7f00',
          1.00, '#ff0000'
        ],
        'heatmap-opacity': ['interpolate',['linear'],['zoom'], 10,0.8, 14,0.6]
      }
    }, 'forte-points')

    // красные «ядра» на большом зуме
    map!.addLayer({
      id:'heat-dots', type:'circle', source:'heat', minzoom:15,
      paint:{
        'circle-radius': ['interpolate',['linear'],['zoom'], 15,2, 18,6],
        'circle-color': '#ff0000',
        'circle-opacity': ['interpolate',['linear'],['zoom'], 15,0.25, 18,0.65]
      }
    })

    // hover handlers
    map!.on('mousemove','forte-points',(e:any)=>{
      map!.getCanvas().style.cursor='pointer'
      const f=e.features?.[0]; if(f) hover.value={x:e.point.x+12,y:e.point.y+12,props:f.properties}
    })
    map!.on('mouseleave','forte-points',()=>{ map!.getCanvas().style.cursor=''; hover.value=null })
    map!.on('mousemove','competitor-points',(e:any)=>{
      map!.getCanvas().style.cursor='pointer'
      const f=e.features?.[0]; if(f) hover.value={x:e.point.x+12,y:e.point.y+12,props:f.properties}
    })
    map!.on('mouseleave','competitor-points',()=>{ map!.getCanvas().style.cursor=''; hover.value=null })

    updateSources()
    applyVisibility()
  })
})

watch([() => props.forteAtms, () => props.competitorAtms, () => props.flows, () => props.heatPoints], updateSources, { deep:true })
watch(() => props.showHeatmap, applyVisibility)

onBeforeUnmount(()=>{ map?.remove(); map=null })
</script>

<style scoped>
.map{ width:100%; height:70vh; border-radius:12px; overflow:hidden; box-shadow:0 10px 30px rgba(0,0,0,.25); }
.tooltip{
  position:absolute; z-index:5; background:rgba(11,18,35,.95); color:#fff;
  border:1px solid #1f2a44; border-radius:10px; padding:10px 12px; font-size:13px;
  transform:translate(-8px,-8px); pointer-events:none;
}
.tooltip .name{ font-weight:700; margin-bottom:6px; }
.tooltip .row{ display:flex; gap:6px; color:#d1d5db; }
.tooltip .row span{ color:#94a3b8; }
</style>
