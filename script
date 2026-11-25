/* ==========================================================
   Noodle House – script.js (ใช้ร่วมกันทุกหน้า)
   - product.html : แสดงสินค้า + ปุ่มกรอง
   - order.html   : ฟอร์มสั่งซื้อ ส่งไป Google Apps Script
   - admin.html   : ตารางคำสั่งซื้อจาก Google Sheets (CSV)
   ========================================================== */
'use strict';

/* ---------- 0. ตั้งค่า (ต้องแก้ 2 ค่านี้ก่อนใช้งานจริง) ---------- */

// URL ของ Google Apps Script Web App (ลงท้ายด้วย /exec)
const APPS_SCRIPT_URL = 'https://script.google.com/macros/s/AKfycbyxzrRI6tJWZbIoH13bRKSdsI6fLAwW0cB9aq9c3-qhuNgHoB9hl4ELeE1SuYZxWWa85w/exec';

// URL CSV ของ Sheet "Orders" (File > Share > Publish to web > เลือก Orders > CSV)
const CSV_URL = 'PASTE_ORDERS_CSV_URL_HERE';

const PRODUCTS_JSON = 'products.json';
const LOW_STOCK_LIMIT = 10; // ต่ำกว่าหรือเท่ากับนี้ แสดงป้ายสต็อกใกล้หมด

/* ---------- 1. ฟังก์ชันช่วยทั่วไป ---------- */

const $ = (id) => document.getElementById(id);

// สร้าง element พร้อม class และข้อความ (ใช้ textContent ป้องกัน XSS)
function el(tag, className, text) {
  const node = document.createElement(tag);
  if (className) node.className = className;
  if (text !== undefined) node.textContent = text;
  return node;
}

function formatBaht(value) {
  const n = Number(value);
  if (!Number.isFinite(n)) return '-';
  return '฿' + n.toLocaleString('th-TH', { maximumFractionDigits: 2 });
}

function isConfigured(url) {
  return typeof url === 'string' && url.startsWith('http') && !url.includes('PASTE_');
}

// สินค้าพร้อมขาย = สถานะ Available และสต็อกมากกว่า 0
function isAvailable(product) {
  return String(product.status).trim().toLowerCase() === 'available' &&
    Number(product.stock) > 0;
}

async function loadProducts() {
  const res = await fetch(PRODUCTS_JSON, { cache: 'no-cache' });
  if (!res.ok) throw new Error('โหลด products.json ไม่สำเร็จ: ' + res.status);
  return res.json();
}

const IMAGE_FALLBACK =
  'data:image/svg+xml;utf8,' +
  encodeURIComponent(
    '<svg xmlns="http://www.w3.org/2000/svg" width="400" height="400">' +
    '<rect width="100%" height="100%" fill="#fff1b8"/>' +
    '<text x="50%" y="50%" font-size="28" text-anchor="middle" fill="#7a6a5c">Noodle House</text></svg>'
  );

/* ==========================================================
   2. หน้า product.html
   ต้องมี: #productGrid และปุ่มกรองที่มี data-filter="all|Available|Out of Stock"
   ========================================================== */

let allProducts = [];

async function initProductsPage() {
  const grid = $('productGrid');
  if (!grid) return;

  grid.replaceChildren(el('div', 'loading', 'กำลังโหลดสินค้า...'));

  try {
    allProducts = await loadProducts();
  } catch (err) {
    console.error(err);
    grid.replaceChildren(el('div', 'error-message', 'ไม่สามารถโหลดข้อมูลสินค้าได้ กรุณาลองใหม่อีกครั้ง'));
    return;
  }

  const buttons = document.querySelectorAll('[data-filter]');
  buttons.forEach((btn) => {
    btn.addEventListener('click', () => {
      buttons.forEach((b) => {
        const active = b === btn;
        b.classList.toggle('btn--outline', !active);
        b.setAttribute('aria-pressed', String(active));
      });
      renderProducts(btn.dataset.filter);
    });
  });

  renderProducts('all');
}

function renderProducts(filter) {
  const grid = $('productGrid');
  if (!grid) return;

  const list = allProducts.filter((p) => {
    if (filter === 'Available') return isAvailable(p);
    if (filter === 'Out of Stock') return !isAvailable(p);
    return true;
  });

  if (list.length === 0) {
    grid.replaceChildren(el('div', 'empty-state', 'ไม่พบสินค้าในหมวดนี้'));
    return;
  }
  grid.replaceChildren(...list.map(createProductCard));
}

function createProductCard(p) {
  const available = isAvailable(p);
  const stock = Number(p.stock) || 0;

  const card = el('article', 'product-card' + (available ? '' : ' product-card--soldout'));

  // รูปสินค้า
  const imgWrap = el('div', 'product-card__image');
  const img = document.createElement('img');
  img.src = p.image;
  img.alt = p.productName;
  img.loading = 'lazy';
  img.addEventListener('error', () => { img.src = IMAGE_FALLBACK; }, { once: true });
  imgWrap.appendChild(img);
  if (!available) imgWrap.appendChild(el('span', 'product-card__badge', 'สินค้าหมด'));

  // เนื้อหา
  const body = el('div', 'product-card__body');
  body.appendChild(el('div', 'product-card__sku', p.sku));
  body.appendChild(el('h3', 'product-card__name', p.productName));
  body.appendChild(el('p', 'product-card__desc', p.description));

  const meta = el('div', 'product-card__meta');
  meta.appendChild(el('span', 'product-card__price', formatBaht(p.price)));
  const stockText = available ? 'คงเหลือ ' + stock + ' ชิ้น' : 'สินค้าหมด';
  const lowClass = !available || stock <= LOW_STOCK_LIMIT ? ' product-card__stock--low' : '';
  meta.appendChild(el('span', 'product-card__stock' + lowClass, stockText));
  body.appendChild(meta);

  // ปุ่มสั่งซื้อ
  if (available) {
    const params = new URLSearchParams({ sku: p.sku, item: p.productName, price: p.price });
    const link = el('a', 'btn btn--block', 'สั่งซื้อ');
    link.href = 'order.html?' + params.toString();
    body.appendChild(link);
  } else {
    const btn = el('button', 'btn btn--block', 'สินค้าหมด');
    btn.type = 'button';
    btn.disabled = true;
    body.appendChild(btn);
  }

  card.append(imgWrap, body);
  return card;
}

/* ==========================================================
   3. หน้า order.html
   id ที่ใช้: orderForm, customerName, contact, address, subdistrict,
   district, province, postalCode, items, quantity, total, note
   ========================================================== */

const order = { sku: '', item: '', price: 0, maxStock: Infinity, valid: false };
let isSubmitting = false;

async function initOrderPage() {
  const form = $('orderForm');
  if (!form) return;

  // 3.1 อ่านค่าจาก URL parameter แล้วเติมลงฟอร์มทันที
  const params = new URLSearchParams(window.location.search);
  order.sku = (params.get('sku') || '').trim();
  order.item = (params.get('item') || '').trim();
  order.price = Number(params.get('price')) || 0;

  const itemsInput = $('items');
  const totalInput = $('total');
  const qtyInput = $('quantity');

  if (itemsInput) { itemsInput.value = order.item; itemsInput.readOnly = true; }
  if (totalInput) { totalInput.readOnly = true; }   // ห้ามลูกค้าแก้ยอดรวมเอง
  if (qtyInput) {
    qtyInput.min = '1';
    qtyInput.step = '1';
    if (!qtyInput.value) qtyInput.value = '1';
    qtyInput.addEventListener('input', updateTotal);
    qtyInput.addEventListener('change', updateTotal);
  }
  updateTotal();

  // 3.2 ตรวจสอบกับ products.json (ไม่เชื่อราคาจาก URL อย่างเดียว)
  try {
    const products = await loadProducts();
    const product = products.find((p) => p.sku === order.sku);
    if (!product || !isAvailable(product)) {
      blockOrder('ไม่พบสินค้านี้หรือสินค้าหมด กรุณากลับไปเลือกสินค้าใหม่');
    } else {
      order.item = product.productName;
      order.price = Number(product.price);
      order.maxStock = Number(product.stock);
      order.valid = true;
      if (itemsInput) itemsInput.value = order.item;
      if (qtyInput) qtyInput.max = String(order.maxStock);
      updateTotal();
    }
  } catch (err) {
    console.error(err);
    blockOrder('ไม่สามารถตรวจสอบข้อมูลสินค้าได้ กรุณาลองใหม่อีกครั้ง');
  }

  form.addEventListener('submit', handleOrderSubmit);
}

function blockOrder(message) {
  order.valid = false;
  const submitBtn = document.querySelector('#orderForm [type="submit"]');
  if (submitBtn) submitBtn.disabled = true;
  alert(message);
}

// คำนวณยอดรวม = ราคาต่อชิ้น x จำนวน
function getQuantity() {
  const q = parseInt($('quantity') ? $('quantity').value : '1', 10);
  return Number.isInteger(q) && q > 0 ? q : 0;
}

function calcTotal() {
  return Math.round(order.price * getQuantity() * 100) / 100;
}

function updateTotal() {
  const totalInput = $('total');
  if (totalInput) totalInput.value = calcTotal();
}

function getFormValues() {
  const val = (id) => ($(id) ? $(id).value.trim() : '');
  return {
    customerName: val('customerName'),
    contact: val('contact'),
    address: val('address'),
    subdistrict: val('subdistrict'),
    district: val('district'),
    province: val('province'),
    postalCode: val('postalCode'),
    note: val('note')
  };
}

// คืนค่า { id, message } ของข้อผิดพลาดแรก หรือ null ถ้าผ่านทั้งหมด
function validateOrder(v) {
  if (!order.valid) return { id: 'items', message: 'ข้อมูลสินค้าไม่ถูกต้อง กรุณาเลือกสินค้าใหม่' };
  if (v.customerName.length < 2) return { id: 'customerName', message: 'กรุณากรอกชื่อลูกค้า' };
  if (!/^0\d{8,9}$/.test(v.contact.replace(/[\s-]/g, ''))) {
    return { id: 'contact', message: 'กรุณากรอกเบอร์โทรศัพท์ให้ถูกต้อง (9-10 หลัก ขึ้นต้นด้วย 0)' };
  }
  if (v.address.length < 5) return { id: 'address', message: 'กรุณากรอกที่อยู่' };
  if (!v.subdistrict) return { id: 'subdistrict', message: 'กรุณากรอกตำบล/แขวง' };
  if (!v.district) return { id: 'district', message: 'กรุณากรอกอำเภอ/เขต' };
  if (!v.province) return { id: 'province', message: 'กรุณากรอกจังหวัด' };
  if (!/^\d{5}$/.test(v.postalCode)) return { id: 'postalCode', message: 'รหัสไปรษณีย์ต้องเป็นตัวเลข 5 หลัก' };
  const qty = getQuantity();
  if (qty < 1) return { id: 'quantity', message: 'กรุณาระบุจำนวนสินค้าอย่างน้อย 1 ชิ้น' };
  if (qty > order.maxStock) {
    return { id: 'quantity', message: 'จำนวนเกินสินค้าคงเหลือ (เหลือ ' + order.maxStock + ' ชิ้น)' };
  }
  return null;
}

function handleOrderSubmit(event) {
  event.preventDefault();
  if (isSubmitting) return; // กันกดซ้ำ

  const values = getFormValues();
  const error = validateOrder(values);
  if (error) {
    alert(error.message);
    if ($(error.id)) $(error.id).focus();
    return;
  }
  if (!isConfigured(APPS_SCRIPT_URL)) {
    alert('ยังไม่ได้ตั้งค่า APPS_SCRIPT_URL ใน script.js');
    return;
  }

  updateTotal(); // คำนวณใหม่อีกครั้งก่อนส่ง

  const payload = {
    customerName: values.customerName,
    contact: values.contact.replace(/[\s-]/g, ''),
    address: values.address,
    subdistrict: values.subdistrict,
    district: values.district,
    province: values.province,
    postalCode: values.postalCode,
    items: order.item,
    sku: order.sku,
    quantity: getQuantity(),
    total: calcTotal(),
    note: values.note
  };

  const submitBtn = document.querySelector('#orderForm [type="submit"]');
  const originalText = submitBtn ? submitBtn.textContent : '';
  isSubmitting = true;
  if (submitBtn) { submitBtn.disabled = true; submitBtn.textContent = 'กำลังส่งคำสั่งซื้อ...'; }

  // ไม่ตั้ง headers และไม่ใช้ mode: 'no-cors'
  fetch(APPS_SCRIPT_URL, {
    method: 'POST',
    body: JSON.stringify(payload)
  })
    .then(() => {
      window.location.href = 'thankyou.html';
    })
    .catch((error) => {
      console.error(error);
      alert('เกิดข้อผิดพลาด กรุณาลองใหม่อีกครั้ง');
      isSubmitting = false;
      if (submitBtn) { submitBtn.disabled = false; submitBtn.textContent = originalText; }
    });
}

/* ==========================================================
   4. หน้า admin.html  (ตาราง #ordersTable tbody)
   ========================================================== */

// parse CSV เอง: รองรับเครื่องหมายคำพูด, comma และขึ้นบรรทัดใหม่ในช่อง, "" แทน "
function parseCSV(text) {
  const rows = [];
  let row = [];
  let field = '';
  let inQuotes = false;

  if (text.charCodeAt(0) === 0xFEFF) text = text.slice(1); // ตัด BOM

  for (let i = 0; i < text.length; i++) {
    const ch = text[i];
    if (inQuotes) {
      if (ch === '"') {
        if (text[i + 1] === '"') { field += '"'; i++; }
        else inQuotes = false;
      } else {
        field += ch;
      }
    } else if (ch === '"') {
      inQuotes = true;
    } else if (ch === ',') {
      row.push(field); field = '';
    } else if (ch === '\n' || ch === '\r') {
      if (ch === '\r' && text[i + 1] === '\n') i++;
      row.push(field); field = '';
      rows.push(row); row = [];
    } else {
      field += ch;
    }
  }
  if (field !== '' || row.length > 0) { row.push(field); rows.push(row); }
  return rows;
}

// แปลง CSV เป็น array ของ object ตามชื่อหัวคอลัมน์ (ข้ามแถวว่างก่อนหัวตาราง)
function csvToObjects(text) {
  const rows = parseCSV(text);
  const headerIndex = rows.findIndex((r) =>
    r.some((c) => c.trim().toLowerCase() === 'order id'));
  if (headerIndex === -1) return [];

  const headers = rows[headerIndex].map((h) => h.trim().toLowerCase());
  return rows
    .slice(headerIndex + 1)
    .filter((r) => r.some((c) => c.trim() !== ''))
    .map((r, i) => {
      const obj = { _index: i };
      headers.forEach((h, c) => { obj[h] = (r[c] || '').trim(); });
      return obj;
    });
}

// แปลงวันที่ รองรับ d/m/yyyy hh:mm:ss และรูปแบบ ISO
function parseOrderDate(str) {
  if (!str) return null;
  const m = str.match(/^(\d{1,2})\/(\d{1,2})\/(\d{4})(?:[ ,T]+(\d{1,2}):(\d{2})(?::(\d{2}))?)?/);
  if (m) {
    let year = Number(m[3]);
    if (year > 2400) year -= 543; // ปี พ.ศ.
    return new Date(year, Number(m[2]) - 1, Number(m[1]),
      Number(m[4] || 0), Number(m[5] || 0), Number(m[6] || 0));
  }
  const d = new Date(str);
  return isNaN(d.getTime()) ? null : d;
}

async function initAdminPage() {
  const tbody = document.querySelector('#ordersTable tbody');
  if (!tbody) return;

  const showRow = (text, cls) => {
    const tr = document.createElement('tr');
    const td = el('td', cls, text);
    td.colSpan = 9;
    td.style.textAlign = 'center';
    tr.appendChild(td);
    tbody.replaceChildren(tr);
  };

  if (!isConfigured(CSV_URL)) {
    showRow('ยังไม่ได้ตั้งค่า CSV_URL ใน script.js', 'error-message');
    return;
  }
  showRow('กำลังโหลดคำสั่งซื้อ...', 'loading');

  try {
    // ใส่ timestamp กัน cache เพื่อให้ได้ข้อมูลล่าสุด
    const url = new URL(CSV_URL);
    url.searchParams.set('_t', Date.now());
    const res = await fetch(url.toString());
    if (!res.ok) throw new Error('โหลด CSV ไม่สำเร็จ: ' + res.status);

    const orders = csvToObjects(await res.text());
    if (orders.length === 0) { showRow('ยังไม่มีคำสั่งซื้อ', 'empty-state'); return; }

    // เรียงล่าสุดขึ้นก่อน (ถ้าอ่านวันที่ไม่ได้ ใช้ลำดับแถวแทน)
    orders.sort((a, b) => {
      const da = parseOrderDate(a['order date']);
      const db = parseOrderDate(b['order date']);
      if (da && db && da.getTime() !== db.getTime()) return db - da;
      return b._index - a._index;
    });

    tbody.replaceChildren(...orders.map(createOrderRow));
  } catch (err) {
    console.error(err);
    showRow('ไม่สามารถโหลดคำสั่งซื้อได้ กรุณาลองใหม่อีกครั้ง', 'error-message');
  }
}

function createOrderRow(o) {
  const date = parseOrderDate(o['order date']);
  const dateText = date ? date.toLocaleString('th-TH') : o['order date'];

  const fullAddress = [
    o['address'],
    o['subdistrict'] && 'ต.' + o['subdistrict'],
    o['district'] && 'อ.' + o['district'],
    o['province'] && 'จ.' + o['province'],
    o['postal code']
  ].filter(Boolean).join(' ');

  const cells = [
    dateText,
    o['order id'],
    o['customer name'],
    o['phone'],
    fullAddress,
    o['product name'],
    o['quantity'],
    o['total'] === '' ? '-' : formatBaht(o['total'].replace(/,/g, '')),
    o['order status']
  ];

  const tr = document.createElement('tr');
  cells.forEach((text) => tr.appendChild(el('td', '', text)));
  return tr;
}

/* ==========================================================
   5. เริ่มทำงานหลังโหลดหน้า (แต่ละฟังก์ชันตรวจ element ของหน้าตัวเองก่อน)
   ========================================================== */

document.addEventListener('DOMContentLoaded', () => {
  initProductsPage();
  initOrderPage();
  initAdminPage();
});
