<template>
    <div class="page">
        <div class="wrap">
            <header>
                <h1>水最美 - 線上滷味下單</h1>
                <p><span class="diamond" aria-hidden="true"></span>如訂單稍多時，請耐心等候。</p>
                <p><span class="diamond" aria-hidden="true"></span>目前皆自取，當場付款。</p>
                <p><span class="diamond" aria-hidden="true"></span>找不到品項時表示已售完。</p>
                <p><span class="diamond" aria-hidden="true"></span>備註未填寫，皆附竹籤一支。</p>
                <div class="img"></div>
            </header>
    
            <main class="card" role="region" aria-labelledby="form-title">
                <h2 id="form-title" class="sr-only">下單表單</h2>
                <form @submit.prevent="submitOrder">
                    <!-- 客戶資訊 -->
                    <section class="grid grid-2" aria-labelledby="cust-info">
                        <h3 id="cust-info" class="sr-only">客戶資訊</h3>
    
                        <div>
                            <label for="name">姓名 (必填)</label>
                            <input id="name" type="text" v-model.trim="form.name" placeholder="請輸入您的姓名" required autocomplete="name" />
                        </div>
    
                        <div>
                            <label for="phone">電話 (必填)</label>
                            <input id="phone" type="tel" v-model.trim="form.phone" inputmode="tel" placeholder="09xxxxxxxx" required autocomplete="tel" />
                            <div class="hint">僅作聯繫用途</div>
                        </div>
    
                        <div>
                        <label for="phone">取貨日期/ 時間 (必填)</label>
                            <div class="grid grid-2">
                                <input id="pickup-date" type="date" v-model="form.pickupDate" :min="todayStr" required aria-label="取餐日期" />
    
                                <!-- 改為兩個下拉：時、分 -->
                                <div class="grid grid-2" aria-label="取餐時間">
                                    <select id="pickup-hour" v-model="form.pickupHour" required aria-label="取餐時" :disabled="hourOptions.length === 0" class="time-select">
          <option disabled value="">時</option>
          <option v-for="h in hourOptions" :key="h" :value="h">{{ h }}</option>
        </select>
    
                                    <select id="pickup-minute" v-model="form.pickupMinute" required aria-label="取餐分" :disabled="!form.pickupHour || minuteOptions.length === 0" class="time-select">
          <option disabled value="">分</option>
          <option v-for="m in minuteOptions" :key="m" :value="m">{{ m }}</option>
        </select>
                                </div>
                            </div>
                            <div class="hint">營業時段 11:00–21:00，僅提供 15 分鐘區段</div>
    
                        </div>
    
    
    
    
    
    
                        <div>
                            <label for="note">備註（選填）</label>
                            <input id="note" type="text" v-model.trim="form.note" placeholder="大辣、小辣、不要竹籤...等等" autocomplete="off" />
                        </div>
                    </section>
    
                    <!-- 菜單清單 -->
                    <section class="menu" aria-labelledby="menu-title">
                        <div class="menu-header">
                            <h3 id="menu-title">滷味品項</h3>
                            <span class="chip" aria-live="polite">合計：{{ totalQty }} 份</span>
                        </div>
    
                        <div class="item" v-for="(m, idx) in menu" :key="m.id">
                            <div>
                                {{ m.name }}
                                <small class="price">NT.{{ fmt(m.price) }}</small>
                            </div>
                            <div class="controls" :aria-label="`調整 ${m.name} 份數`">
                                <button type="button" class="btn icon" @click="dec(idx)" :aria-label="`減少 ${m.name}`">-</button>
                                <div class="qty" aria-live="polite">{{ m.qty }}</div>
                                <button type="button" class="btn icon" @click="inc(idx)" :aria-label="`增加 ${m.name}`">+</button>
                            </div>
                        </div>
    
                        <div class="total">
                            合計：{{ totalQty }} 份 / NT.{{ fmt(totalPrice) }} 元 ( <strong>取餐付款 </strong> )
                        </div>
                    </section>
    
                    <!-- 送出 -->
                    <div class="actions">
                        <p id="submit-hint" class="sr-only">請先填寫姓名、電話，選日期與時間，並至少加入一份品項。</p>
    
                        <button type="submit" class="btn primary send" :disabled="!canSubmit" aria-describedby="submit-hint">
            送出訂單
          </button>
                    </div>
                </form>
            </main>
    
            <!-- 成功 Modal（未串 API 也可用） -->
            <div class="overlay" :class="{ show: showModal }" @click.self="closeModal" role="dialog" aria-labelledby="modal-title">
                <div class="modal" role="document">
                    <h3 id="modal-title">下單成功 ✅</h3>
                    <p class="mb">
                        感謝訂購！您的取貨單號為電話後 6 碼 :<br> # <span class="receveNum">298710</span><br> 取餐時間：
                        <strong>{{ form.pickupAt }}</strong><br> 合計：
                        <strong>{{ totalQty }}</strong> 份 / <strong>NT.{{ fmt(totalPrice) }}</strong> 元（取餐付款）
                    </p>
    
                    <div v-if="summaryItems.length" class="scroll">
                        <ul>
                            <li v-for="it in summaryItems" :key="it.id">
                                {{ it.name }} × {{ it.qty }} ＝ NT.{{ fmt(it.subtotal) }}
                            </li>
                        </ul>
                    </div>
    
                    <div class="modal-actions">
                        <button class="btn" @click="closeModal">關閉</button>
                    </div>
                </div>
            </div>
        </div>
    </div>
</template>
<script setup>
import { computed, reactive, ref, watch } from 'vue'

/** 以本地時區取得 YYYY-MM-DD（避免 UTC 跨日問題） */
function getTodayStrLocal() {
  const d = new Date()
  const y = d.getFullYear()
  const m = String(d.getMonth() + 1).padStart(2, '0')
  const day = String(d.getDate()).padStart(2, '0')
  return `${y}-${m}-${day}`
}
const todayStr = getTodayStrLocal()

/* ====== 表單狀態 ====== */
const form = reactive({
  name: '',
  phone: '',
  pickupDate: todayStr,   // 預設今天
  // 時分下拉（新增）
  pickupHour: '',         // '11'..'21'
  pickupMinute: '',       // '00' | '15' | '30' | '45'
  // 舊欄位：保留讓總結與驗證沿用
  pickupTime: '',         // 'HH:mm'
  pickupAt: '',           // 'YYYY-MM-DD HH:mm'
  note: ''
})

/* ====== 菜單 ====== */
const menu = ref([
  { id: 'seaweed', name: '海帶',     price: 30, qty: 0 },
  { id: 'tofu',    name: '豆干',     price: 35, qty: 0 },
  { id: 'intestine', name: '大腸',   price: 80, qty: 0 },
  { id: 'egg',     name: '滷蛋',     price: 15, qty: 0 },
  { id: 'wing',    name: '雞翅',     price: 60, qty: 0 },
  { id: 'big-tofu', name: '大豆干',  price: 40, qty: 0 },
  { id: 'meatball', name: '貢丸',    price: 35, qty: 0 }
])

const showModal = ref(false)

/* ====== 小計、合計 ====== */
const totalQty = computed(() =>
  menu.value.reduce((s, i) => s + (Number(i.qty) || 0), 0)
)
const totalPrice = computed(() =>
  menu.value.reduce((s, i) => s + ((Number(i.qty) || 0) * (i.price || 0)), 0)
)

/* ====== 數量調整與格式化 ====== */
function inc(idx) { menu.value[idx].qty++ }
function dec(idx) { menu.value[idx].qty = Math.max(0, (Number(menu.value[idx].qty) || 0) - 1) }
function fmt(n) { try { return Number(n).toLocaleString('zh-TW') } catch { return n } }

const summaryItems = computed(() =>
  menu.value
    .filter(m => (Number(m.qty) || 0) > 0)
    .map(m => ({ id: m.id, name: m.name, qty: Number(m.qty) || 0, subtotal: (Number(m.qty) || 0) * (m.price || 0) }))
)

/* ====== 營業與時間工具 ====== */
const BUSINESS_START = '11:00'
const BUSINESS_END   = '21:00'
const STEP_MINUTES   = 15
const LEAD_MINUTES   = 0   // 若需「今天需晚於現在 N 分鐘」，改 >0 如 20

function hhmmToMin(hhmm) {
  const [h, m] = hhmm.split(':').map(Number)
  return h * 60 + m
}
function minToHhmm(mins) {
  const h = String(Math.floor(mins / 60)).padStart(2, '0')
  const m = String(mins % 60).padStart(2, '0')
  return `${h}:${m}`
}
function nowPlusLeadHHmm() {
  const d = new Date()
  d.setMinutes(d.getMinutes() + LEAD_MINUTES)
  return minToHhmm(d.getHours() * 60 + d.getMinutes())
}

function makePickupAt(dateStr, timeStr) {
  if (!dateStr || !timeStr) return ''
  if (!/^\d{2}:\d{2}$/.test(timeStr)) return ''
  return `${dateStr} ${timeStr}`
}

/* ====== 下拉選項：時、分 ====== */
const hourOptions = computed(() => {
  const startH = Number(BUSINESS_START.split(':')[0]) // 11
  const endH   = Number(BUSINESS_END.split(':')[0])   // 21
  const arr = []
  for (let h = startH; h <= endH; h++) arr.push(String(h).padStart(2, '0'))

  // 若今天且需要緩衝，過濾掉現在之前的小時
  if (form.pickupDate === todayStr && LEAD_MINUTES > 0) {
    const minH = Number(nowPlusLeadHHmm().split(':')[0])
    return arr.filter(h => Number(h) >= minH)
  }
  return arr
})

const minuteOptions = computed(() => {
  if (!form.pickupHour) return []
  const h = Number(form.pickupHour)
  const mins = [0, 15, 30, 45].map(n => String(n).padStart(2, '0'))

  // 21 點只提供 00 分（避免超出 21:00）
  if (h === Number(BUSINESS_END.split(':')[0])) return ['00']

  // 若今天且需要緩衝：該小時需過濾當下分鐘之前
  if (form.pickupDate === todayStr && LEAD_MINUTES > 0) {
    const [minH, minM] = nowPlusLeadHHmm().split(':').map(Number)
    if (h < minH) return []
    if (h > minH) return mins
    return mins.filter(m => Number(m) >= minM)
  }

  return mins
})

/* ====== 範圍驗證（沿用你的 canSubmit） ====== */
function isTimeInBusinessRange(timeStr) {
  if (!/^\d{2}:\d{2}$/.test(timeStr)) return false
  const t = hhmmToMin(timeStr)
  const inRange = t >= hhmmToMin(BUSINESS_START) && t <= hhmmToMin(BUSINESS_END)
  if (!inRange) return false
  if (LEAD_MINUTES > 0 && form.pickupDate === todayStr) {
    return t >= hhmmToMin(nowPlusLeadHHmm())
  }
  return true
}

/* ====== 將時、分組回 HH:mm；生成 pickupAt ====== */
watch(
  () => [form.pickupDate, form.pickupHour, form.pickupMinute],
  () => {
    // 時或分未選齊 → 清空
    if (!form.pickupHour || !form.pickupMinute) {
      form.pickupTime = ''
      form.pickupAt   = ''
      return
    }
    const hhmm = `${form.pickupHour}:${form.pickupMinute}`

    // 分選項因小時切換而失效的情況 → 重置
    if (!minuteOptions.value.includes(form.pickupMinute)) {
      form.pickupMinute = ''
      form.pickupTime = ''
      form.pickupAt   = ''
      return
    }

    form.pickupTime = hhmm
    form.pickupAt   = makePickupAt(form.pickupDate, hhmm)
  },
  { immediate: true }
)

/* ====== 送出條件 ====== */
const canSubmit = computed(() => {
  const nameOk = form.name.trim().length > 0
  const phoneOk = /^\d[\d\-\s]{7,14}$/.test(form.phone.trim())
  const qtyOk = totalQty.value > 0
  const dateOk = !!form.pickupDate && form.pickupDate >= todayStr
  const timeOk = isTimeInBusinessRange(form.pickupTime)
  const pickupOk = !!form.pickupAt
  return nameOk && phoneOk && qtyOk && dateOk && timeOk && pickupOk
})

/* ====== 送出 / 關閉 ====== */
function submitOrder() {
  if (!canSubmit.value) return
  showModal.value = true
}
function closeModal() { showModal.value = false }

/* ====== 清除：原生 time picker 相關（已刪除 pickupTimeEl / openTimePicker） ====== */
</script>
