<script setup>
import { ref, computed, onMounted } from 'vue'

// ── State ────────────────────────────────────────────────────────
const products      = ref([])       // รายการสินค้าทั้งหมดจาก API
const loading       = ref(true)     // แสดง loading spinner
const search        = ref('')       // ข้อความค้นหา
const catFilter     = ref('')       // category ที่กำลัง filter
const showModal     = ref(false)    // แสดง/ซ่อน modal เพิ่ม/แก้ไข
const editingId     = ref(null)     // null = กำลัง add, มีค่า = กำลัง edit
const confirmDelete = ref(null)     // เก็บ product ที่จะลบ (ใช้ confirm dialog)

// ข้อมูลในฟอร์ม modal
const form = ref({ name: '', category: '', price: '', stock: 0, description: '' })

let debounceTimer = null            // สำหรับ debounce search input
// ── Computed ─────────────────────────────────────────────────────

// กรองสินค้าตาม search + catFilter พร้อมกัน
const filtered = computed(() => {
  const s = search.value.toLowerCase()
  return products.value.filter(p => {
    // ตรงกับ search text? (เช็คทั้ง name และ description)
    const matchS = !s ||
      p.name.toLowerCase().includes(s) ||
      (p.description || '').toLowerCase().includes(s)
    // ตรงกับ category filter?
    const matchC = !catFilter.value || p.category === catFilter.value
    return matchS && matchC
  })
})

// สร้าง list ของ categories จาก products ที่มีอยู่ (unique + sort)
const categories = computed(() =>
  [...new Set(products.value.map(p => p.category))].sort()
)

// คำนวณ 4 ตัวเลข dashboard stats
const stats = computed(() => ({
  total:      products.value.length,
  lowStock:   products.value.filter(p => p.stock < 10).length,
  totalItems: products.value.reduce((s, p) => s + p.stock, 0),
  totalValue: products.value.reduce((s, p) => s + parseFloat(p.price) * p.stock, 0)
}))

// ── Methods ───────────────────────────────────────────────────────

// debounce: รอ 280ms หลัง user หยุดพิมพ์แล้วค่อย fetch
// (ป้องกัน API ถูกเรียกทุก keystroke)
function debounceFetch() {
  clearTimeout(debounceTimer)
  debounceTimer = setTimeout(fetchProducts, 280)
}

// ดึงสินค้าจาก API (ส่ง search + catFilter เป็น query params)
async function fetchProducts() {
  loading.value = true
  const q = new URLSearchParams()
  if (search.value)    q.set('search',   search.value)
  if (catFilter.value) q.set('category', catFilter.value)
  try {
    const res = await fetch(`/api/products?${q}`)
    products.value = await res.json()
  } catch {
    products.value = []
  } finally {
    loading.value = false
  }
}

// เปิด modal แบบ Add (clear form)
function openAdd() {
  editingId.value = null
  form.value = { name: '', category: '', price: '', stock: 0, description: '' }
  showModal.value = true
}

// เปิด modal แบบ Edit (copy ข้อมูลจาก product เข้า form)
function openEdit(p) {
  editingId.value = p.id
  form.value = {
    name: p.name, category: p.category,
    price: p.price, stock: p.stock,
    description: p.description || ''
  }
  showModal.value = true
}

// บันทึก (ถ้า editingId มีค่า = PUT, ถ้าเป็น null = POST)
async function saveProduct() {
  const body = {
    name:        form.value.name,
    category:    form.value.category,
    price:       parseFloat(form.value.price),
    stock:       parseInt(form.value.stock),
    description: form.value.description
  }
  const url    = editingId.value ? `/api/products/${editingId.value}` : '/api/products'
  const method = editingId.value ? 'PUT' : 'POST'
  await fetch(url, { method, headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(body) })
  showModal.value = false
  fetchProducts()   // reload สินค้าใหม่
}

// ลบสินค้า
async function deleteProduct(id) {
  await fetch(`/api/products/${id}`, { method: 'DELETE' })
  confirmDelete.value = null
  fetchProducts()
}

// คืน CSS class ตาม stock level (ใช้กับ stock bar และตัวเลข)
function stockClass(s) {
  if (s <= 0) return 'out'    // หมด
  if (s < 10) return 'low'    // ใกล้หมด
  if (s < 30) return 'mid'    // ปานกลาง
  return 'high'               // เพียงพอ
}

// คืน CSS class ตาม category (ใช้กับ badge สี)
function catClass(cat) {
  const m = {
    Electronics: 'c-elec', Clothing: 'c-cloth',
    Footwear:    'c-foot',  Food:     'c-food',
    Sports:      'c-sport', Books:    'c-book',
    Furniture:   'c-furn',  Bags:     'c-bag',
    Accessories: 'c-acc',   Tools:    'c-tool'
  }
  return m[cat] || 'c-other'
}

// โหลดสินค้าทันทีเมื่อ component mount ครั้งแรก
onMounted(fetchProducts)
</script>

<template>
  <div>

    <!-- HEADER -->
    <header class="app-header">
      <div class="logo">
        <span class="logo-icon">📦</span>
        <div>
          <div class="logo-name">Dark Web</div>
          <div class="logo-sub">ระบบจัดการสินค้ามืดคงคลัง</div>
        </div>
      </div>
      <button class="btn-add" @click="openAdd">+ เพิ่มสินค้า</button>
    </header>

    <main class="main">

      <!-- STATS DASHBOARD -->
      <div class="stats-grid">
        <div class="stat-card">
          <div class="stat-icon si-green">📦</div>
          <div class="stat-body">
            <div class="stat-val stat-val--total">{{ stats.total }}</div>
            <div class="stat-label">สินค้าทั้งหมด</div>
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-icon si-red">⚠️</div>
          <div class="stat-body">
            <div class="stat-val stat-val--low">{{ stats.lowStock }}</div>
            <div class="stat-label">สต็อกใกล้หมด (<10)</div>
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-icon si-amber">📊</div>
          <div class="stat-body">
            <div class="stat-val stat-val--items">{{ stats.totalItems.toLocaleString() }}</div>
            <div class="stat-label">จำนวนสต็อกรวม</div>
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-icon si-blue">💰</div>
          <div class="stat-body">
            <div class="stat-val stat-val--value">
              ฿{{ Math.round(stats.totalValue).toLocaleString() }}
            </div>
            <div class="stat-label">มูลค่าสต็อกรวม</div>
          </div>
        </div>
      </div>

      <!-- LOW STOCK ALERT -->
      <div class="alert-low" v-if="stats.lowStock > 0">
        🚨 มีสินค้า <strong>{{ stats.lowStock }} รายการ</strong>
        ที่มีจำนวนสต็อกน้อยกว่า 10 ชิ้น — กรุณาตรวจสอบและเติมสต็อก
      </div>

      <!-- TOOLBAR -->
      <div class="toolbar">
        <input
          v-model="search"
          @input="debounceFetch"
          class="input-search"
          placeholder="🔍 ค้นหาชื่อสินค้า หรือคำอธิบาย..."
        />
        <select v-model="catFilter" @change="fetchProducts" class="input-select">
          <option value="">ทุกหมวดหมู่</option>
          <option v-for="c in categories" :key="c" :value="c">{{ c }}</option>
        </select>
        <span class="result-count" v-if="!loading">
          แสดง {{ filtered.length }} / {{ products.length }} รายการ
        </span>
      </div>
            <!-- LOADING -->
      <div class="state-box" v-if="loading">
        <div class="state-icon">⏳</div>
        <div>กำลังโหลดข้อมูลสินค้า...</div>
      </div>

      <!-- EMPTY -->
      <div class="state-box" v-else-if="filtered.length === 0">
        <div class="state-icon">📭</div>
        <div class="state-title">ไม่พบสินค้า</div>
        <div class="state-desc">
          {{ search || catFilter ? 'ลองเปลี่ยน keyword หรือ filter' : 'กดปุ่ม "+ เพิ่มสินค้า" เพื่อเริ่มต้น' }}
        </div>
      </div>

      <!-- PRODUCT GRID -->
      <div class="product-grid" v-else>
        <div
          v-for="p in filtered"
          :key="p.id"
          class="product-card"
          :class="{ 'card-low': p.stock > 0 && p.stock < 10, 'card-out': p.stock <= 0 }"
        >
          <div class="card-top">
            <span class="cat-badge" :class="catClass(p.category)">{{ p.category }}</span>
            <div class="product-name">{{ p.name }}</div>
            <div class="product-desc" v-if="p.description">{{ p.description }}</div>
            <div class="product-price">
              ฿{{ parseFloat(p.price).toLocaleString('th-TH', { minimumFractionDigits: 2 }) }}
            </div>
            <div class="stock-info">
              <div class="stock-row">
                <span class="stock-label">สต็อก</span>
                <span class="stock-num" :class="stockClass(p.stock)">
                  {{ p.stock.toLocaleString() }} ชิ้น
                  <span v-if="p.stock <= 0"> — หมดแล้ว!</span>
                  <span v-else-if="p.stock < 10"> — ใกล้หมด!</span>
                </span>
              </div>
              <div class="stock-track">
                <div
                  class="stock-fill"
                  :class="stockClass(p.stock)"
                  :style="{ width: Math.min((p.stock / 80) * 100, 100) + '%' }"
                ></div>
              </div>
            </div>
          </div>
          <div class="card-footer">
            <button class="btn-edit" @click="openEdit(p)">✏️ แก้ไข</button>
            <button class="btn-del" @click="confirmDelete = p">🗑️ ลบ</button>
          </div>
        </div>
      </div>

    </main>
        <!-- ADD/EDIT MODAL -->
    <div class="overlay" v-if="showModal" @click.self="showModal = false">
      <div class="modal">
        <div class="modal-title">
          {{ editingId ? '✏️ แก้ไขสินค้า' : '📦 เพิ่มสินค้าใหม่' }}
        </div>
        <form @submit.prevent="saveProduct">
          <div class="form-group">
            <label class="form-label">ชื่อสินค้า *</label>
            <input class="form-input" v-model="form.name" placeholder="เช่น MacBook Air M3" required />
          </div>
          <div class="form-row">
            <div class="form-group">
              <label class="form-label">หมวดหมู่ *</label>
              <input class="form-input" v-model="form.category" list="cat-list" placeholder="เช่น Electronics" required />
              <datalist id="cat-list">
                <option v-for="c in categories" :value="c" :key="c" />
              </datalist>
            </div>
            <div class="form-group">
              <label class="form-label">ราคา (บาท) *</label>
              <input class="form-input" v-model="form.price" type="number" min="0" step="0.01" placeholder="0.00" required />
            </div>
          </div>
          <div class="form-group">
            <label class="form-label">จำนวนสต็อก (ชิ้น) *</label>
            <input class="form-input" v-model="form.stock" type="number" min="0" placeholder="0" required />
          </div>
          <div class="form-group">
            <label class="form-label">คำอธิบายสินค้า</label>
            <textarea class="form-input" v-model="form.description" rows="3" placeholder="รายละเอียดเพิ่มเติม (ถ้ามี)" style="resize:vertical"></textarea>
          </div>
          <div class="modal-footer">
            <button type="button" class="btn-cancel" @click="showModal = false">ยกเลิก</button>
            <button type="submit" class="btn-save">
              {{ editingId ? 'บันทึกการเปลี่ยนแปลง' : 'เพิ่มสินค้า' }}
            </button>
          </div>
        </form>
      </div>
    </div>

    <!-- DELETE CONFIRM -->
    <div class="overlay" v-if="confirmDelete" @click.self="confirmDelete = null">
      <div class="modal confirm">
        <div class="confirm-icon">🗑️</div>
        <div class="confirm-title">ยืนยันการลบสินค้า</div>
        <div class="confirm-desc">
          คุณต้องการลบ <strong>{{ confirmDelete?.name }}</strong> ออกจากระบบหรือไม่?<br>
          <span style="color:#f87171">การกระทำนี้ไม่สามารถย้อนกลับได้</span>
        </div>
        <div class="confirm-actions">
          <button class="btn-cancel" @click="confirmDelete = null">ยกเลิก</button>
          <button class="btn-danger-confirm" @click="deleteProduct(confirmDelete.id)">ลบเลย</button>
        </div>
      </div>
    </div>

  </div>
</template>

<style scoped>
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

:root, div {
  --bg:        #0a0a0e;
  --bg-elev:   #131318;
  --bg-card:   #16161d;
  --border:    #2a2a35;
  --border-glow: #3f1d2e;
  --text:      #e4e4ec;
  --text-dim:  #8b8b9c;
  --accent:    #b91c1c;
  --accent-glow: #ef4444;
  --accent2:   #7e22ce;
}

div { color: var(--text); }

.app-header {
  position: sticky; top: 0; z-index: 100;
  background: linear-gradient(180deg, #14141b 0%, #0d0d12 100%);
  border-bottom: 1px solid var(--border-glow);
  height: 62px; padding: 0 1.5rem;
  display: flex; align-items: center; gap: .85rem;
  box-shadow: 0 2px 12px rgba(0,0,0,.6), 0 1px 0 rgba(239,68,68,.08);
}
.logo { display: flex; align-items: center; gap: .6rem; }
.logo-icon {
  font-size: 1.6rem;
  filter: drop-shadow(0 0 6px rgba(239,68,68,.5));
}
.logo-name {
  font-weight: 800; font-size: 1.15rem; line-height: 1;
  color: #ff4d4d;
  text-shadow: 0 0 10px rgba(239,68,68,.6), 0 0 2px rgba(239,68,68,.8);
  letter-spacing: .5px;
}
.logo-sub  { font-size: .72rem; color: var(--text-dim); }
.btn-add {
  margin-left: auto;
  background: linear-gradient(135deg, #7f1d1d, #b91c1c);
  color: #fff;
  border: 1px solid #ef4444;
  border-radius: 8px;
  padding: .55rem 1.2rem; font-size: .9rem; font-weight: 700;
  cursor: pointer; transition: all .2s;
  box-shadow: 0 0 10px rgba(239,68,68,.35);
}
.btn-add:hover {
  background: linear-gradient(135deg, #991b1b, #ef4444);
  box-shadow: 0 0 18px rgba(239,68,68,.6);
}

.main {
  max-width: 1280px; margin: 0 auto; padding: 1.75rem 1.5rem;
  background: var(--bg);
  min-height: 100vh;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1rem; margin-bottom: 1.5rem;
}
.stat-card {
  background: linear-gradient(160deg, var(--bg-card), #0f0f14);
  border: 1px solid var(--border);
  border-radius: 12px;
  padding: 1.1rem; display: flex; align-items: center; gap: .9rem;
  box-shadow: inset 0 1px 0 rgba(255,255,255,.02), 0 4px 14px rgba(0,0,0,.5);
}
.stat-icon {
  width: 46px; height: 46px; border-radius: 10px;
  display: flex; align-items: center; justify-content: center;
  font-size: 1.3rem; flex-shrink: 0;
  border: 1px solid var(--border);
}
.si-green { background: rgba(34,197,94,.08); box-shadow: 0 0 12px rgba(34,197,94,.15); }
.si-red   { background: rgba(239,68,68,.1); box-shadow: 0 0 12px rgba(239,68,68,.25); }
.si-amber { background: rgba(245,158,11,.08); box-shadow: 0 0 12px rgba(245,158,11,.15); }
.si-blue  { background: rgba(139,92,246,.1); box-shadow: 0 0 12px rgba(139,92,246,.2); }
.stat-val { font-size: 1.65rem; font-weight: 800; line-height: 1; }
.stat-val--total { color: #4ade80; }
.stat-val--low   { color: #f87171; text-shadow: 0 0 8px rgba(248,113,113,.5); }
.stat-val--items { color: #fbbf24; }
.stat-val--value { color: #c084fc; font-size: 1.3rem; }
.stat-label { font-size: .8rem; color: var(--text-dim); margin-top: .2rem; }

.alert-low {
  background: rgba(127,29,29,.25);
  border: 1px solid #7f1d1d;
  border-radius: 10px; padding: .85rem 1.2rem;
  font-size: .92rem; margin-bottom: 1.5rem; color: #fca5a5;
  box-shadow: 0 0 16px rgba(239,68,68,.15), inset 0 0 20px rgba(127,29,29,.2);
}

.toolbar { display: flex; gap: .75rem; margin-bottom: 1.5rem; flex-wrap: wrap; align-items: center; }
.input-search {
  flex: 1; min-width: 200px;
  padding: .6rem 1rem; border: 1.5px solid var(--border); border-radius: 8px;
  font-size: .95rem; outline: none; transition: border-color .2s, box-shadow .2s;
  background: var(--bg-elev); color: var(--text);
}
.input-search::placeholder { color: var(--text-dim); }
.input-search:focus { border-color: var(--accent-glow); box-shadow: 0 0 10px rgba(239,68,68,.25); }
.input-select {
  padding: .6rem .9rem; border: 1.5px solid var(--border); border-radius: 8px;
  background: var(--bg-elev); color: var(--text); font-size: .9rem; outline: none; cursor: pointer;
}
.input-select:focus { border-color: var(--accent-glow); }
.result-count { font-size: .82rem; color: var(--text-dim); white-space: nowrap; }

.state-box { text-align: center; padding: 4rem 1rem; color: var(--text-dim); }
.state-icon { font-size: 3rem; margin-bottom: .75rem; filter: grayscale(1) opacity(.7); }
.state-title { font-size: 1.1rem; font-weight: 700; color: var(--text); margin-bottom: .3rem; }
.state-desc { font-size: .9rem; }

.product-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(275px, 1fr));
  gap: 1.2rem;
}
.product-card {
  background: linear-gradient(165deg, var(--bg-card), #101015);
  border: 1px solid var(--border);
  border-radius: 14px;
  overflow: hidden; transition: transform .2s, box-shadow .2s, border-color .2s;
}
.product-card:hover {
  transform: translateY(-3px);
  box-shadow: 0 10px 28px rgba(0,0,0,.7), 0 0 14px rgba(239,68,68,.12);
  border-color: #44303a;
}
.product-card.card-low  { border-color: #7f1d1d; box-shadow: 0 0 14px rgba(239,68,68,.18); }
.product-card.card-out  { border-color: #2a2a35; opacity: .55; filter: grayscale(.4); }

.card-top { padding: 1.1rem; }
.cat-badge {
  display: inline-block; padding: .2rem .65rem;
  border-radius: 20px; font-size: .72rem; font-weight: 700; margin-bottom: .6rem;
  border: 1px solid rgba(255,255,255,.06);
}
.c-elec  { background: rgba(59,130,246,.12); color: #93c5fd; }
.c-cloth { background: rgba(236,72,153,.12); color: #f9a8d4; }
.c-foot  { background: rgba(168,85,247,.12); color: #d8b4fe; }
.c-food  { background: rgba(245,158,11,.12); color: #fcd34d; }
.c-sport { background: rgba(34,197,94,.12); color: #86efac; }
.c-book  { background: rgba(56,189,248,.12); color: #7dd3fc; }
.c-furn  { background: rgba(192,132,252,.12); color: #e9d5ff; }
.c-bag   { background: rgba(251,146,60,.12); color: #fdba74; }
.c-acc   { background: rgba(74,222,128,.12); color: #bbf7d0; }
.c-tool  { background: rgba(148,163,184,.12); color: #cbd5e1; }
.c-other { background: rgba(167,139,250,.12); color: #ddd6fe; }

.product-name  { font-size: 1rem; font-weight: 700; color: var(--text); margin-bottom: .3rem; line-height: 1.35; }
.product-desc  { font-size: .82rem; color: var(--text-dim); line-height: 1.5; margin-bottom: .75rem; }
.product-price { font-size: 1.25rem; font-weight: 800; color: #4ade80; text-shadow: 0 0 8px rgba(74,222,128,.25); }

.stock-info { margin-top: .85rem; }
.stock-row  { display: flex; justify-content: space-between; font-size: .82rem; margin-bottom: .3rem; }
.stock-label { color: var(--text-dim); }
.stock-num   { font-weight: 700; }
.stock-num.out  { color: #6b7280; }
.stock-num.low  { color: #f87171; text-shadow: 0 0 6px rgba(248,113,113,.4); }
.stock-num.mid  { color: #fbbf24; }
.stock-num.high { color: #4ade80; }
.stock-track { height: 6px; background: #1f1f28; border-radius: 3px; overflow: hidden; }
.stock-fill  { height: 100%; border-radius: 3px; transition: width .4s ease; min-width: 4px; }
.stock-fill.out  { background: #3a3a45; width: 2% !important; }
.stock-fill.low  { background: #ef4444; box-shadow: 0 0 6px rgba(239,68,68,.6); }
.stock-fill.mid  { background: #f59e0b; }
.stock-fill.high { background: #22c55e; }

.card-footer {
  display: flex; gap: .5rem;
  padding: .75rem 1.1rem;
  border-top: 1px solid var(--border); background: #0d0d12;
}
.btn-edit, .btn-del {
  flex: 1; padding: .45rem; border-radius: 7px;
  font-size: .82rem; font-weight: 600; cursor: pointer; border: 1px solid var(--border); transition: all .2s;
}
.btn-edit { background: #1a1a22; color: #cbd5e1; }
.btn-edit:hover { background: #232330; border-color: #3f3f4d; }
.btn-del  { background: rgba(127,29,29,.2); color: #f87171; border-color: #7f1d1d; }
.btn-del:hover  { background: rgba(127,29,29,.4); box-shadow: 0 0 10px rgba(239,68,68,.25); }

.overlay {
  position: fixed; inset: 0;
  background: rgba(0,0,0,.75);
  backdrop-filter: blur(2px);
  display: flex; align-items: center; justify-content: center;
  z-index: 500; padding: 1rem;
}
.modal {
  background: linear-gradient(160deg, #16161d, #0e0e13);
  border: 1px solid var(--border-glow);
  border-radius: 16px;
  width: 100%; max-width: 480px;
  max-height: 90vh; overflow-y: auto; padding: 2rem;
  box-shadow: 0 0 40px rgba(0,0,0,.8), 0 0 20px rgba(239,68,68,.08);
}
.modal-title { font-size: 1.2rem; font-weight: 700; margin-bottom: 1.5rem; color: var(--text); }

.form-group { margin-bottom: 1rem; }
.form-label { display: block; font-size: .87rem; font-weight: 600; margin-bottom: .35rem; color: var(--text-dim); }
.form-input {
  width: 100%; padding: .6rem .85rem;
  border: 1.5px solid var(--border); border-radius: 8px;
  font-size: .95rem; outline: none; transition: border-color .2s, box-shadow .2s;
  background: var(--bg-elev); color: var(--text);
}
.form-input::placeholder { color: #5b5b6b; }
.form-input:focus { border-color: var(--accent-glow); box-shadow: 0 0 10px rgba(239,68,68,.2); }
.form-row { display: grid; grid-template-columns: 1fr 1fr; gap: .75rem; }

.modal-footer {
  display: flex; gap: .75rem; justify-content: flex-end;
  padding-top: 1rem; border-top: 1px solid var(--border); margin-top: 1rem;
}
.btn-cancel {
  background: #1a1a22; color: #cbd5e1;
  border: 1px solid var(--border); border-radius: 8px; padding: .6rem 1.25rem;
  font-weight: 600; cursor: pointer; transition: background .2s;
}
.btn-cancel:hover { background: #232330; }
.btn-save {
  background: linear-gradient(135deg, #15803d, #22c55e);
  color: #06150c;
  border: 1px solid #4ade80; border-radius: 8px; padding: .6rem 1.25rem;
  font-weight: 700; cursor: pointer; transition: all .2s;
  box-shadow: 0 0 10px rgba(34,197,94,.25);
}
.btn-save:hover { box-shadow: 0 0 18px rgba(34,197,94,.45); }

.confirm { text-align: center; max-width: 380px; }
.confirm-icon  { font-size: 3rem; margin-bottom: 1rem; filter: drop-shadow(0 0 8px rgba(239,68,68,.4)); }
.confirm-title { font-size: 1.1rem; font-weight: 700; margin-bottom: .5rem; color: var(--text); }
.confirm-desc  { font-size: .9rem; color: var(--text-dim); margin-bottom: 1.5rem; line-height: 1.6; }
.confirm-actions { display: flex; gap: .75rem; justify-content: center; }
.btn-danger-confirm {
  background: linear-gradient(135deg, #7f1d1d, #dc2626);
  color: #fff;
  border: 1px solid #ef4444; border-radius: 8px; padding: .6rem 1.4rem;
  font-weight: 700; cursor: pointer; transition: all .2s;
  box-shadow: 0 0 10px rgba(239,68,68,.35);
}
.btn-danger-confirm:hover { box-shadow: 0 0 20px rgba(239,68,68,.6); }

@media (max-width: 640px) {
  .form-row { grid-template-columns: 1fr; }
  .stats-grid { grid-template-columns: 1fr 1fr; }
}

/* --- Interactive enhancements: subtle animations and button effects --- */
.product-card { will-change: transform, opacity; }
.product-card { animation: fadeUp .36s cubic-bezier(.2,.9,.3,1) both; }
@keyframes fadeUp {
  from { opacity: 0; transform: translateY(6px); }
  to   { opacity: 1; transform: translateY(0); }
}

/* Shared button base for nicer interactions */
.btn-edit, .btn-del, .btn-cancel, .btn-save, .btn-add, .btn-danger-confirm {
  position: relative;
  overflow: hidden;
  transition: transform .12s ease, box-shadow .18s ease, filter .12s ease;
}
.btn-edit:active, .btn-del:active, .btn-cancel:active, .btn-save:active, .btn-add:active, .btn-danger-confirm:active {
  transform: translateY(1px) scale(.997);
  filter: brightness(.98);
}
.btn-edit:focus-visible, .btn-del:focus-visible, .btn-cancel:focus-visible, .btn-save:focus-visible, .btn-add:focus-visible, .btn-danger-confirm:focus-visible {
  outline: 2px solid rgba(196, 181, 253, .12);
  outline-offset: 3px;
}

/* Gentle hover lift for primary controls */
.btn-add:hover { transform: translateY(-3px) scale(1.02); }

/* Small ripple effect using pseudo-element centered on the button (simple) */
.btn-edit::after, .btn-del::after, .btn-cancel::after, .btn-save::after, .btn-add::after, .btn-danger-confirm::after {
  content: '';
  position: absolute; inset: 0; border-radius: inherit;
  background: radial-gradient(circle at 50% 50%, rgba(255,255,255,0.06), rgba(255,255,255,0.02) 30%, transparent 40%);
  opacity: 0; pointer-events: none; transition: opacity .24s ease, transform .24s ease;
  transform: scale(.96);
}
.btn-edit:active::after, .btn-del:active::after, .btn-cancel:active::after, .btn-save:active::after, .btn-add:active::after, .btn-danger-confirm:active::after {
  opacity: 1; transform: scale(1);
}

/* Emphasize low-stock items with a subtle pulse */
.product-card.card-low { animation: lowPulse 2.2s ease-in-out infinite; }
@keyframes lowPulse {
  0% { box-shadow: 0 10px 24px rgba(0,0,0,.65), 0 0 10px rgba(239,68,68,.12); }
  50% { box-shadow: 0 14px 30px rgba(0,0,0,.68), 0 0 20px rgba(239,68,68,.14); }
  100% { box-shadow: 0 10px 24px rgba(0,0,0,.65), 0 0 10px rgba(239,68,68,.12); }
}

</style>