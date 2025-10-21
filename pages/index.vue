<template>
  <div class="app">
    <header class="hdr">
      <strong>Forte GeoSmart</strong>
      <small>Интеллектуальный навигатор сети ATM</small>
    </header>

    <!-- Фильтры -->
    <section class="filters">
      <label>
        Район
        <select v-model="district">
          <option :value="null">Все</option>
          <option v-for="d in districts" :key="d" :value="d">{{ d }}</option>
        </select>
      </label>

      <label class="toggle">
        <input type="checkbox" v-model="showHeatmap" />
        Показать Heatmap (трафик)
      </label>

      <button @click="reset">Сбросить</button>
    </section>

    <!-- Основная сетка -->
    <main class="grid">
      <!-- Карта -->
      <section class="card map">
        <ClientOnly>
          <MapView
            :forte-atms="filteredForte"
            :competitor-atms="filteredCompetitors"
            :flows="filteredFlows"
            :heat-points="heatPoints"
            :show-heatmap="showHeatmap"
          />
        </ClientOnly>
      </section>

      <!-- Правая панель -->
      <aside class="card side">
        <div class="metric">
          <div class="label">Средняя загруженность</div>
          <div class="value">{{ (avgLoad * 100).toFixed(0) }}%</div>
          <small>по частоте снятий</small>
        </div>

        <div class="metric">
          <div class="label">Простаивающие средства</div>
          <div class="value">{{ fmtKZT(idleFunds) }}</div>
          <small>остаток × downtime%</small>
        </div>

        <div class="metric">
          <div class="label">Инкассационные маршруты</div>
          <div class="value">{{ filteredFlows.length }}</div>
          <small>активные связи ATM</small>
        </div>

        <div class="insights">
          <div class="title">Рекомендации (AI Insight)</div>
          <ul>
            <li v-for="r in recommendations" :key="r.text">
              {{ r.text }}
              <div class="bar"><div :style="{ width: r.impact * 100 + '%' }" /></div>
            </li>
          </ul>
        </div>
      </aside>
    </main>
  </div>
</template>

<script setup lang="ts">
import { ref, computed } from 'vue'
import MapView from '~/components/MapView.vue'

type ATM = { id:number; name:string; lat:number; lng:number; isForte:boolean; district:string;
  avgCashBalanceKZT?:number; withdrawalFreqPerDay?:number; downtimePct?:number; bank?:string }
type Flow = { fromId:number; toId:number; weight:number }
type HeatPoint = { lat:number; lng:number; weight:number }

// --- Мок-данные ---
const forteATMs = ref<ATM[]>([
  { id:101, name:'ATM #101', district:'Есиль',    lat:51.128, lng:71.43,  isForte:true,  avgCashBalanceKZT:18_000_000, withdrawalFreqPerDay:320, downtimePct:0.03 },
  { id:213, name:'ATM #213', district:'Сарыарка', lat:51.155, lng:71.41,  isForte:true,  avgCashBalanceKZT:9_600_000,  withdrawalFreqPerDay:110, downtimePct:0.12 },
  { id:318, name:'ATM #318', district:'Есиль',    lat:51.170, lng:71.47,  isForte:true,  avgCashBalanceKZT:13_400_000, withdrawalFreqPerDay:210, downtimePct:0.05 },
  { id:402, name:'ATM #402', district:'Байконур', lat:51.145, lng:71.50,  isForte:true,  avgCashBalanceKZT:11_200_000, withdrawalFreqPerDay:160, downtimePct:0.07 }
])
const competitorATMs = ref<ATM[]>([
  { id:9001, name:'KZ Bank ATM',   district:'Есиль',    lat:51.125, lng:71.44, isForte:false, bank:'KZ Bank' },
  { id:9002, name:'Alma Bank ATM', district:'Сарыарка', lat:51.160, lng:71.405,isForte:false, bank:'Alma' }
])
const flows = ref<Flow[]>([
  { fromId:101, toId:213, weight:0.6 },
  { fromId:213, toId:318, weight:0.35 },
  { fromId:318, toId:402, weight:0.45 },
  { fromId:402, toId:101, weight:0.25 }
])

const districts = ['Есиль','Сарыарка','Алатау','Байконур','Туран']
const district = ref<string|null>(null)
const showHeatmap = ref<boolean>(false)

// фильтрация
const filteredForte = computed(() => forteATMs.value.filter(a => !district.value || a.district === district.value))
const filteredCompetitors = computed(() => competitorATMs.value.filter(a => !district.value || a.district === district.value))
const filteredFlows = computed(() => {
  const ids = new Set(filteredForte.value.map(a => a.id))
  return flows.value.filter(f => ids.has(f.fromId) && ids.has(f.toId))
})

// точки для heatmap (вес берём из частоты снятий, нормируем 0..1)
const heatPoints = computed<HeatPoint[]>(() =>
  filteredForte.value.map(a => ({
    lat: a.lat,
    lng: a.lng,
    weight: Math.min((a.withdrawalFreqPerDay ?? 0) / 400, 1)
  }))
)

// метрики/инсайты
const avgLoad = computed(() => {
  const arr = filteredForte.value
  if (!arr.length) return 0
  return arr.reduce((s,a)=>s+(a.withdrawalFreqPerDay||0)/400,0)/arr.length
})
const idleFunds = computed(() =>
  filteredForte.value.reduce((s,a)=>s+(a.avgCashBalanceKZT||0)*(a.downtimePct||0),0)
)
const recommendations = [
  { text:'Переместить ATM #213 → ЖК Highvill (↑ трафик +32%)', impact:0.32 },
  { text:'Добавить ATM в район Expo (нет конкурентов)', impact:0.25 }
]

function reset(){ district.value=null; showHeatmap.value=false }
function fmtKZT(n:number){
  return new Intl.NumberFormat('ru-KZ',{style:'currency',currency:'KZT',maximumFractionDigits:0}).format(n)
}
</script>

<style scoped>
.app{background:#0a0f1f;color:#e5e7eb;min-height:100vh;padding:16px;}
.hdr{display:flex;justify-content:space-between;align-items:center;
     border-bottom:1px solid #1f2a44;padding-bottom:10px;margin-bottom:14px;}
.filters{display:flex;gap:12px;align-items:center;margin-bottom:12px;flex-wrap:wrap}
.filters select{background:#0b1223;color:#e5e7eb;border:1px solid #1f2a44;
                padding:8px 10px;border-radius:10px;}
.filters .toggle{display:flex;gap:8px;align-items:center}
.filters button{background:#0b1223;color:#e5e7eb;border:1px solid #1f2a44;
                padding:8px 12px;border-radius:10px;cursor:pointer;}
.grid{display:grid;grid-template-columns:1fr 320px;gap:16px;}
.card{background:rgba(255,255,255,0.03);border:1px solid #1f2a44;
      border-radius:16px;padding:12px;}
.map{padding:0;}
.side{display:flex;flex-direction:column;gap:12px;}
.metric{background:#0b1223;border:1px solid #1f2a44;border-radius:12px;
        padding:12px;display:flex;flex-direction:column;gap:4px;}
.metric .label{color:#94a3b8;font-size:13px;}
.metric .value{font-size:26px;font-weight:700;}
.insights{margin-top:8px;background:#0b1223;border:1px solid #1f2a44;
          border-radius:12px;padding:12px;}
.insights .title{font-weight:600;margin-bottom:8px;}
.insights ul{list-style:none;margin:0;padding:0;display:flex;flex-direction:column;gap:10px;}
.bar{height:6px;background:#1f2a44;border-radius:6px;margin-top:4px;overflow:hidden;}
.bar div{height:100%;background:#7c3aed;}
</style>
