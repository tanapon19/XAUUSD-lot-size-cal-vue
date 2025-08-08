<template>
  <div class="container">
    <div class="header">
      <div class="title">
        <div class="badge"><span class="badge-icon">🥇</span> XAUUSD • Lot Calculator (v2)</div>
        <h1>โปรแกรมคำนวนขนาด Lot</h1>
        <div class="sub">ใส่ <b>Entry / SL / TP</b> เป็นราคา + ตั้งค่า Risk (% หรือ $) แล้วระบบคำนวณ Lot ให้ พร้อมแชร์ลิงก์และจำค่าล่าสุด</div>
      </div>
      <div class="toolbar">
        <div class="pill">💾 <strong>Auto‑save</strong></div>
        <div class="pill">🔗 <strong>Shareable</strong></div>
        <div class="pill">⚡ <strong>Instant</strong></div>
      </div>
    </div>

    <div class="card">
      <div class="grid">
        <div>
          <label>คู่เทรด</label>
          <select class="select" disabled>
            <option>XAUUSD (ทองคำ)</option>
          </select>
        </div>
        <div>
          <label>สกุลเงินพอร์ต</label>
          <select class="select" disabled>
            <option>USD</option>
          </select>
        </div>

        <div>
          <label>Balance (USD)</label>
          <input class="input" v-model.number="balance" type="number" min="0" step="0.01" placeholder="1000" />
        </div>
        <div class="row">
          <div>
            <label>Risk (%)</label>
            <input class="input" v-model.number="riskPct" type="number" min="0" step="0.01" placeholder="1" />
          </div>
          <div>
            <label>หรือ Risk $ (ถ้ามี)</label>
            <input class="input" v-model="riskFixed" type="number" min="0" step="0.01" placeholder="10" />
          </div>
        </div>

        <div class="row">
          <div>
            <label>Entry</label>
            <input class="input" v-model="entry" type="number" min="0" step="0.01" placeholder="2400.00" />
          </div>
          <div>
            <label>SL</label>
            <input class="input" v-model="sl" type="number" min="0" step="0.01" placeholder="2388.00" />
          </div>
          <div>
            <label>TP</label>
            <input class="input" v-model="tp" type="number" min="0" step="0.01" placeholder="2424.00" />
          </div>
        </div>

        <div class="row">
          <div>
            <label>Contract size ต่อ 1 lot (XAUUSD)</label>
            <input class="input" v-model.number="contract" type="number" min="1" step="1" />
            <div class="help">ส่วนใหญ่: 1 lot = 100 oz (โปรดตรวจสอบกับโบรกเกอร์)</div>
          </div>
          <div>
            <label>Lot ขั้นต่ำ / step</label>
            <input class="input" v-model.number="lotStep" type="number" min="0.01" step="0.01" />
            <div class="help">ระบบจะปัดลงตาม step เพื่อให้เปิดออเดอร์ได้จริง</div>
          </div>
        </div>
      </div>

      <div class="actions">
        <button class="btn btn-primary" @click="fillDemo">ตัวอย่าง</button>
        <button class="btn btn-ghost" @click="resetAll">ล้างค่า</button>
        <button class="btn btn-soft" @click="copyShareLink">คัดลอกลิงก์แชร์</button>
      </div>

      <div class="kpis">
        <div class="kpi">
          <div class="label">🎯 Lot Size (แนะนำ)</div>
          <div class="value">{{ fmt(lot, 2) }}</div>
        </div>
        <div class="kpi">
          <div class="label">📦 Units (oz)</div>
          <div class="value">{{ fmt(units, 2) }}<span v-if="units"> oz</span></div>
        </div>
        <div class="kpi">
          <div class="label">🛡️ ความเสี่ยงต่อออเดอร์</div>
          <div class="value">{{ isFinite$(riskAmount) ? '$ ' + fmt(riskAmount, 2) : '—' }}</div>
        </div>
      </div>

      <div class="hr"></div>

      <div class="kpis">
        <div class="kpi">
          <div class="label">📉 ระยะความเสี่ยง/oz (|Entry − SL|)</div>
          <div class="value">{{ fmt(priceRiskPerUnit, 2) }}</div>
        </div>
        <div class="kpi">
          <div class="label">📈 Risk : Reward (ถ้ามี TP)</div>
          <div class="value">{{ isFinite$(rr) ? '1 : ' + fmt(rr, 2) : '—' }}</div>
        </div>
        <div class="kpi">
          <div class="label">💰 กำไรที่คาด (ถ้า TP)</div>
          <div class="value">{{ isFinite$(pnl.profit) ? '$ ' + fmt(pnl.profit, 2) : '—' }}</div>
        </div>
      </div>

      <div class="hr"></div>

      <div class="help">
        สูตร: <code>lot = risk / (|entry - sl| × contract)</code> •
        แชร์: ปุ่ม “คัดลอกลิงก์แชร์” •
        จำค่าอัตโนมัติ: localStorage
      </div>
    </div>

    <div class="footer">Powered by Day in the Life</div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted } from 'vue'

const KEY = 'xauusd-lot-vue:v2-pretty'

// state
const balance = ref(1000)
const riskPct = ref(1)
const riskFixed = ref('')
const entry = ref('')
const sl = ref('')
const tp = ref('')
const contract = ref(100)
const lotStep = ref(0.01)

// helpers
const fmt = (n, d = 2) => {
  if (n === null || n === undefined || Number.isNaN(n)) return '—'
  return new Intl.NumberFormat('en-US', { minimumFractionDigits: d, maximumFractionDigits: d }).format(n)
}
const floorToStep = (value, step) => {
  if (!isFinite(value) || !isFinite(step) || step <= 0) return value
  const k = Math.floor(value / step) * step
  return Math.max(0, Number(k.toFixed(8)))
}
const isFinite$ = (v) => Number.isFinite(v)

// URL encode/decode
const toQuery = (state) => {
  const p = new URLSearchParams()
  const put = (k, v) => { if (v !== '' && v !== undefined && v !== null) p.set(k, String(v)) }
  put('b', state.balance)
  put('rp', state.riskPct)
  put('rf', state.riskFixed)
  put('e', state.entry)
  put('sl', state.sl)
  put('tp', state.tp)
  put('c', state.contract)
  put('ls', state.lotStep)
  return p.toString()
}
const fromQuery = (qs) => {
  const p = new URLSearchParams(qs)
  const pick = (k) => (p.has(k) ? p.get(k) : undefined)
  const num = (x) => (x === undefined ? undefined : x === '' ? '' : Number(x))
  return {
    balance: num(pick('b')), riskPct: num(pick('rp')), riskFixed: num(pick('rf')),
    entry: num(pick('e')), sl: num(pick('sl')), tp: num(pick('tp')),
    contract: num(pick('c')), lotStep: num(pick('ls')),
  }
}

// boot: localStorage then URL
onMounted(() => {
  try {
    const raw = localStorage.getItem(KEY)
    if (raw) {
      const s = JSON.parse(raw)
      if (s && typeof s === 'object') {
        if (Number.isFinite(s.balance)) balance.value = s.balance
        if (Number.isFinite(s.riskPct)) riskPct.value = s.riskPct
        if (s.riskFixed === '' || Number.isFinite(s.riskFixed)) riskFixed.value = s.riskFixed
        if (s.entry === '' || Number.isFinite(s.entry)) entry.value = s.entry
        if (s.sl === '' || Number.isFinite(s.sl)) sl.value = s.sl
        if (s.tp === '' || Number.isFinite(s.tp)) tp.value = s.tp
        if (Number.isFinite(s.contract)) contract.value = s.contract
        if (Number.isFinite(s.lotStep)) lotStep.value = s.lotStep
      }
    }
  } catch {}

  const q = fromQuery(window.location.search)
  const may = (v, set) => { if (v !== undefined) set(v) }
  may(q.balance, (v) => balance.value = v)
  may(q.riskPct, (v) => riskPct.value = v)
  if (q.riskFixed !== undefined) riskFixed.value = (q.riskFixed === '' ? '' : q.riskFixed)
  may(q.entry, (v) => entry.value = v)
  may(q.sl, (v) => sl.value = v)
  may(q.tp, (v) => tp.value = v)
  may(q.contract, (v) => contract.value = v)
  may(q.lotStep, (v) => lotStep.value = v)
})

// persist localStorage (debounced)
let lsTimer
watch([balance, riskPct, riskFixed, entry, sl, tp, contract, lotStep], () => {
  clearTimeout(lsTimer)
  lsTimer = setTimeout(() => {
    const payload = {
      balance: Number(balance.value),
      riskPct: Number(riskPct.value),
      riskFixed: riskFixed.value === '' ? '' : Number(riskFixed.value),
      entry: entry.value === '' ? '' : Number(entry.value),
      sl: sl.value === '' ? '' : Number(sl.value),
      tp: tp.value === '' ? '' : Number(tp.value),
      contract: Number(contract.value),
      lotStep: Number(lotStep.value),
    }
    try { localStorage.setItem(KEY, JSON.stringify(payload)) } catch {}
  }, 200)
}, { immediate: false })

// sync URL (debounced)
let urlTimer
watch([balance, riskPct, riskFixed, entry, sl, tp, contract, lotStep], () => {
  clearTimeout(urlTimer)
  urlTimer = setTimeout(() => {
    const qs = toQuery({
      balance: balance.value, riskPct: riskPct.value, riskFixed: riskFixed.value,
      entry: entry.value, sl: sl.value, tp: tp.value, contract: contract.value, lotStep: lotStep.value
    })
    const url = qs ? `?${qs}` : window.location.pathname
    window.history.replaceState(null, '', url)
  }, 300)
}, { immediate: false })

// derived
const priceRiskPerUnit = computed(() => {
  const e = parseFloat(entry.value), s = parseFloat(sl.value)
  if (!isFinite(e) || !isFinite(s) || e <= 0 || s <= 0) return null
  const v = Math.abs(e - s)
  return v === 0 ? null : v
})
const riskAmount = computed(() => {
  const rf = parseFloat(riskFixed.value)
  if (isFinite(rf) && rf > 0) return rf
  if (!isFinite(balance.value) || !isFinite(riskPct.value)) return null
  return balance.value * (riskPct.value / 100)
})
const lot = computed(() => {
  const c = parseFloat(contract.value)
  const step = parseFloat(lotStep.value)
  if (!isFinite(c) || c <= 0) return null
  if (!isFinite(riskAmount.value) || !isFinite(priceRiskPerUnit.value)) return null
  const raw = riskAmount.value / (priceRiskPerUnit.value * c)
  return floorToStep(raw, step)
})
const units = computed(() => {
  const c = parseFloat(contract.value)
  if (!isFinite(c) || !isFinite(lot.value)) return null
  return lot.value * c
})
const rr = computed(() => {
  const e = parseFloat(entry.value), t = parseFloat(tp.value)
  if (!isFinite(e) || !isFinite(t) || !isFinite(priceRiskPerUnit.value) || priceRiskPerUnit.value <= 0) return null
  return Math.abs(t - e) / priceRiskPerUnit.value
})
const pnl = computed(() => {
  const e = parseFloat(entry.value), t = parseFloat(tp.value)
  if (!isFinite(e) || !isFinite(t) || !isFinite(units.value)) return { profit: null, loss: null }
  const rewardPerUnit = Math.abs(t - e)
  const riskPerUnit = priceRiskPerUnit.value ?? null
  const profit = rewardPerUnit * units.value
  const loss = isFinite(riskPerUnit) ? riskPerUnit * units.value : null
  return { profit, loss }
})

// actions
const fillDemo = () => {
  balance.value = 1000
  riskPct.value = 1
  riskFixed.value = ''
  entry.value = 2400.00
  sl.value = 2388.00
  tp.value = 2424.00
  contract.value = 100
  lotStep.value = 0.01
}
const resetAll = () => {
  entry.value = ''; sl.value = ''; tp.value = ''; riskFixed.value = ''
}
const copyShareLink = async () => {
  const qs = toQuery({
    balance: balance.value, riskPct: riskPct.value, riskFixed: riskFixed.value,
    entry: entry.value, sl: sl.value, tp: tp.value, contract: contract.value, lotStep: lotStep.value
  })
  const link = `${window.location.origin}${window.location.pathname}${qs ? '?' + qs : ''}`
  try {
    await navigator.clipboard.writeText(link)
    alert('คัดลอกลิงก์แล้ว!')
  } catch {
    prompt('คัดลอกลิงก์ด้วยตนเอง:', link)
  }
}
</script>
