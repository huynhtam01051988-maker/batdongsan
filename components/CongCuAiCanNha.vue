<!--
  Công cụ nhân viên trên trang chi tiết căn nhà: Viết 5 tin AI + Tra quy hoạch.
  - Chỉ hiện khi máy này đã đăng nhập trang Quản Trị (/admin). Khách xem web không thấy.
  - "Viết 5 tin AI" mở công cụ của web Nhà Đất Mr. Tâm (chế độ nhúng ?nhung=1) ngay trên trang,
    điền sẵn thông tin công khai của căn (tiêu đề, địa chỉ, giá, diện tích, kết cấu, pháp lý, mô tả...).
    AI chỉ chạy khi nhân viên đăng nhập công cụ và tự bấm "Viết tin". Khóa AI nằm trên máy chủ AI riêng, không nằm trong file này.
  - "Tra quy hoạch" mở cổng tra cứu của Sở Quy hoạch – Kiến trúc TP.HCM; kết quả ghi vào ô Quy hoạch trong khung Viết 5 tin.
  Dùng: <CongCuAiCanNha :property="property" /> trong pages/bat-dong-san/[id].vue (khung đầy đủ),
        <CongCuAiCanNha :property="property" compact /> trong components/PropertyCard.vue (2 nút gọn dưới mỗi thẻ căn).
  Muốn gỡ: xóa 2 dòng đó và file này.
-->
<template>
  <div v-if="isStaff && property && compact" class="ai-staff-row">
    <button type="button" class="btn btn-gold ai-staff-mini" @click="openTool">
      <i class="fa-solid fa-wand-magic-sparkles"></i> Viết 5 tin AI
    </button>
    <a
      class="btn btn-outline-gold ai-staff-mini"
      :href="QH_URL"
      target="_blank"
      rel="noopener"
      title="Mở cổng tra quy hoạch của Sở; địa chỉ căn được chép sẵn để dán"
      @click="copyAddress"
    >
      <i class="fa-solid fa-map-location-dot"></i> Tra quy hoạch
    </a>
    <small v-if="copied" class="ai-staff-copied">Đã chép địa chỉ căn, dán vào ô Tìm vị trí trên cổng.</small>
  </div>

  <div v-else-if="isStaff && property" class="ai-staff-box">
    <div class="ai-staff-head">
      <span class="ai-staff-title"><i class="fa-solid fa-wand-magic-sparkles"></i> Công cụ nhân viên</span>
      <small class="ai-staff-sub">Chỉ máy đã đăng nhập Quản Trị mới thấy, khách không thấy</small>
    </div>
    <div class="ai-staff-actions">
      <button type="button" class="btn btn-gold" @click="openTool">
        <i class="fa-solid fa-wand-magic-sparkles"></i> Viết 5 tin AI
      </button>
      <a class="btn btn-outline-gold" :href="QH_URL" target="_blank" rel="noopener">
        <i class="fa-solid fa-map-location-dot"></i> Tra quy hoạch (cổng của Sở)
      </a>
      <button type="button" class="btn btn-outline-gold" @click="copyAddress">
        <i class="fa-regular fa-copy"></i> {{ copied ? 'Đã chép địa chỉ' : 'Chép địa chỉ' }}
      </button>
    </div>
    <p class="ai-staff-note">
      Tra quy hoạch: mở cổng của Sở → bấm Tìm vị trí → dán địa chỉ. Ghi kết quả vào ô Quy hoạch trong khung Viết 5 tin để AI đưa vào tin.
    </p>
  </div>

  <Teleport to="body">
    <div v-if="open" class="ai-tool-overlay" @click.self="closeTool">
      <div class="ai-tool-modal" role="dialog" aria-label="Viết 5 tin AI">
        <div class="ai-tool-modal-head">
          <b><i class="fa-solid fa-wand-magic-sparkles"></i> Viết 5 tin AI · {{ property?.id }}</b>
          <button type="button" class="ai-tool-x" aria-label="Đóng" @click="closeTool">
            <i class="fa-solid fa-xmark"></i>
          </button>
        </div>
        <iframe
          ref="frame"
          class="ai-tool-iframe"
          :src="TOOL_URL"
          title="Công cụ viết tin AI"
          allow="clipboard-write"
          @load="sendProperty"
        ></iframe>
        <p class="ai-tool-modal-foot">
          {{ status }}
          <a :href="TOOL_URL" target="_blank" rel="noopener">Mở công cụ ở tab mới</a>
        </p>
      </div>
    </div>
  </Teleport>
</template>

<script setup lang="ts">
import { ref, onMounted, onUnmounted } from 'vue';

const props = defineProps<{ property: any; compact?: boolean }>();

const TOOL_ORIGIN = 'https://nha-dat-mr-tam.pages.dev';
const TOOL_URL = TOOL_ORIGIN + '/?nhung=1';
const QH_URL = 'https://gisxaydung.tphcm.gov.vn/tracuuttqh';
const TAG = 'mrtam-nhung';

const isStaff = ref(false);
const open = ref(false);
const copied = ref(false);
const status = ref('');
const frame = ref<HTMLIFrameElement | null>(null);
let timer: ReturnType<typeof setInterval> | null = null;
let run = '';

onMounted(() => {
  try {
    isStaff.value = !!localStorage.getItem('bds_admin_session');
  } catch {
    isStaff.value = false;
  }
  window.addEventListener('message', onMessage);
});

onUnmounted(() => {
  window.removeEventListener('message', onMessage);
  stopSending();
});

const line = (label: string, value: unknown) => {
  const s = String(value ?? '').trim();
  return s && s !== '0' ? `${label}: ${s}` : '';
};

const featureText = () => {
  const f = props.property?.features;
  try {
    const list = typeof f === 'string' ? JSON.parse(f) : f;
    return Array.isArray(list) ? list.join(', ') : '';
  } catch {
    return '';
  }
};

const addressText = () => {
  const p = props.property || {};
  return [p.location, p.ward, p.province].filter(Boolean).join(', ');
};

// Chỉ gửi thông tin đang hiện công khai trên trang căn nhà (không gửi dữ liệu nội bộ).
const propertyText = () => {
  const p = props.property || {};
  return [
    line('Tiêu đề', p.title),
    line('Địa chỉ', p.location),
    line('Phường', p.ward),
    line('Tỉnh/Thành', p.province),
    line('Giá', p.price),
    line('Diện tích', p.area),
    line('Kích thước', p.dimensions),
    line('Kết cấu', p.structure),
    line('Phòng ngủ', p.bedrooms),
    line('Phòng tắm (WC)', p.bathrooms),
    line('Hướng', p.direction),
    line('Pháp lý', p.legal),
    line('Dòng tiền thuê', p.rentIncome),
    line('Đặc điểm', featureText()),
    line('Mô tả', p.description),
    line('Mã căn', p.id),
  ].filter(Boolean).join('\n');
};

function stopSending() {
  if (timer) {
    clearInterval(timer);
    timer = null;
  }
}

// Công cụ trong khung mới tải xong: gửi thông tin căn, lặp lại tới khi công cụ báo đã nhận.
function sendProperty() {
  stopSending();
  const win = frame.value?.contentWindow;
  if (!win) return;
  run = Date.now().toString(36) + Math.random().toString(36).slice(2, 8);
  const msg = { tk: TAG, type: 'can', v: 1, run, text: propertyText().slice(0, 20000), addr: addressText().slice(0, 300) };
  let tries = 0;
  status.value = 'Đang chuyển thông tin căn sang công cụ…';
  const post = () => {
    tries++;
    try {
      win.postMessage(msg, TOOL_ORIGIN);
    } catch {
      // bỏ qua
    }
    if (tries >= 40) {
      stopSending();
      status.value = 'Công cụ chưa nhận được thông tin căn. Bấm Đóng rồi mở lại.';
    }
  };
  post();
  timer = setInterval(post, 700);
}

function onMessage(ev: MessageEvent) {
  if (ev.origin !== TOOL_ORIGIN) return;
  const d: any = ev.data;
  if (!d || d.tk !== TAG || d.type !== 'got' || d.run !== run) return;
  stopSending();
  status.value = 'Đã chuyển thông tin căn. Lần đầu dùng: đăng nhập nhân viên trong khung.';
}

function openTool() {
  open.value = true;
}

function closeTool() {
  open.value = false;
  stopSending();
  status.value = '';
}

async function copyAddress() {
  try {
    await navigator.clipboard.writeText(addressText());
    copied.value = true;
    setTimeout(() => { copied.value = false; }, 2500);
  } catch {
    copied.value = false;
  }
}
</script>

<style scoped>
.ai-staff-row {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-top: 12px;
  padding-top: 12px;
  border-top: 1px dashed var(--border-gold);
}

.ai-staff-mini {
  flex: 1 1 0;
  min-width: 0;
  justify-content: center;
  white-space: nowrap;
  font-size: 0.8rem;
  padding: 7px 10px;
}

.ai-staff-copied {
  flex-basis: 100%;
  color: var(--text-muted);
  font-size: 0.72rem;
}

.ai-staff-box {
  background: var(--bg-card);
  border: 1px dashed var(--border-gold);
  border-radius: var(--radius-lg);
  padding: 16px 20px;
  margin-bottom: 24px;
}

.ai-staff-head {
  display: flex;
  align-items: baseline;
  flex-wrap: wrap;
  gap: 6px 12px;
  margin-bottom: 12px;
}

.ai-staff-title {
  color: var(--gold-primary);
  font-weight: 700;
  font-size: 0.95rem;
}

.ai-staff-sub {
  color: var(--text-muted);
  font-size: 0.78rem;
}

.ai-staff-actions {
  display: flex;
  flex-wrap: wrap;
  gap: 10px;
}

.ai-staff-actions .btn {
  font-size: 0.85rem;
  padding: 8px 14px;
}

.ai-staff-note {
  margin: 10px 0 0;
  color: var(--text-muted);
  font-size: 0.78rem;
  line-height: 1.5;
}

.ai-tool-overlay {
  position: fixed;
  inset: 0;
  z-index: 5000;
  background: rgba(10, 10, 12, 0.6);
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 12px;
}

.ai-tool-modal {
  width: min(1200px, 100%);
  height: min(92vh, 100%);
  background: var(--bg-card);
  border: 1px solid var(--border-gold);
  border-radius: 12px;
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.ai-tool-modal-head {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  padding: 10px 14px;
  border-bottom: 1px solid var(--border-color);
  color: var(--text-main);
}

.ai-tool-modal-head b i {
  color: var(--gold-primary);
}

.ai-tool-x {
  border: 1px solid var(--border-color);
  background: transparent;
  color: var(--text-main);
  width: 34px;
  height: 34px;
  border-radius: 50%;
  cursor: pointer;
}

.ai-tool-iframe {
  flex: 1;
  width: 100%;
  border: 0;
  background: #fff;
}

.ai-tool-modal-foot {
  margin: 0;
  padding: 8px 14px;
  font-size: 0.8rem;
  color: var(--text-muted);
  border-top: 1px solid var(--border-color);
}

.ai-tool-modal-foot a {
  color: var(--gold-primary);
  font-weight: 600;
  margin-left: 6px;
}
</style>
