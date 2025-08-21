<template>
  <div class="page">
    <div class="wrap">
      <header>
        <h1>水最美 - 線上滷味滷味下單</h1>
        <p><span class="diamond" aria-hidden="true"></span>取餐時間必填，如訂單稍多時，請等待一下。</p>
        <p><span class="diamond" aria-hidden="true"></span>目前皆自取，當場付款。</p>
        <p><span class="diamond" aria-hidden="true"></span>備註未填寫，皆附筷子一雙。</p>
        <div class="img"></div>
      </header>

      <main class="card" role="region" aria-labelledby="form-title">
        <h2 id="form-title" class="sr-only">下單表單</h2>
        <form @submit.prevent="submitOrder">
          <!-- 客戶資訊 -->
          <section class="grid grid-2" aria-labelledby="cust-info">
            <h3 id="cust-info" class="sr-only">客戶資訊</h3>

            <div>
              <label for="name">姓名</label>
              <input id="name" type="text" v-model.trim="form.name"
                     placeholder="請輸入您的姓名" required autocomplete="name" />
            </div>

            <div>
              <label for="phone">電話</label>
              <input id="phone" type="tel" v-model.trim="form.phone" inputmode="tel"
                     placeholder="09xx-xxx-xxx" required autocomplete="tel" />
              <div class="hint">僅作聯繫用途</div>
            </div>

            <div>
              <label for="pickup">取餐時間 (必填)</label>
              <input id="pickup" type="text" v-model="form.pickupAt"
                     placeholder="自由輸入24時製，如 18:30 " required autocomplete="off" />
            </div>

            <div>
              <label for="note">備註（可選）</label>
              <input id="note" type="text" v-model.trim="form.note"
                     placeholder="大辣、小辣、不要餐具..." autocomplete="off" />
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
              合計：{{ totalQty }} 份 / NT.{{ fmt(totalPrice) }} 元 — <strong>取餐付款</strong>
            </div>
          </section>

          <!-- 送出 -->
         <div class="actions">
  <button
    type="submit"
    class="btn primary send"
    :disabled="!canSubmit"
    aria-describedby="submit-hint"
  >
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
            感謝您的訂購！<br>
            取餐時間：<strong>{{ form.pickupAt }}</strong><br>
            合計：<strong>{{ totalQty }}</strong> 份 / <strong>NT.{{ fmt(totalPrice) }}</strong> 元（取餐付款）
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
import { computed, reactive, ref } from 'vue'

const form = reactive({
  name: '',
  phone: '',
  pickupAt: '',
  note: ''
})

const menu = ref([
  { id: 'seaweed',   name: '海帶',   price: 30, qty: 0 },
  { id: 'tofu',      name: '豆干',   price: 35, qty: 0 },
  { id: 'intestine', name: '大腸',   price: 80, qty: 0 },
  { id: 'egg',       name: '滷蛋',   price: 15, qty: 0 },
  { id: 'wing',      name: '雞翅',   price: 60, qty: 0 },
  { id: 'big-tofu',  name: '大豆干', price: 40, qty: 0 },
  { id: 'meatball',  name: '貢丸',   price: 35, qty: 0 }
])

const showModal = ref(false)

const totalQty   = computed(() => menu.value.reduce((s, i) => s + (Number(i.qty) || 0), 0))
const totalPrice = computed(() => menu.value.reduce((s, i) => s + (i.qty * (i.price || 0)), 0))

const canSubmit = computed(() => {
  const basic = form.name.trim().length > 0 && form.phone.trim().length > 0 && form.pickupAt && totalQty.value > 0
  const phoneOk = /^\d[\d\-\s]{7,14}$/.test(form.phone.trim())
  return basic && phoneOk
})

function inc(idx) { menu.value[idx].qty++ }
function dec(idx) { menu.value[idx].qty = Math.max(0, menu.value[idx].qty - 1) }

function fmt(n) {
  try { return Number(n).toLocaleString('zh-TW') } catch { return n }
}

const summaryItems = computed(() =>
  menu.value.filter(m => m.qty > 0).map(m => ({
    id: m.id, name: m.name, qty: m.qty, subtotal: m.qty * m.price
  }))
)

function submitOrder () {
  if (!canSubmit.value) return
  // 暫未串 API：直接顯示成功彈窗
  showModal.value = true
}

function closeModal () { showModal.value = false }
</script>

