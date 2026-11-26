<!--
  Trang "Công cụ viết tin AI" cho web Bến Thành Land (repo ngo-tuankiet/batdongsan).
  Chỉ THÊM file này vào thư mục pages/ (pages/cong-cu-ai.vue). Không sửa file cũ nào.
  Địa chỉ trang: benthanhland.com/cong-cu-ai. Không có trong menu; Google không đưa lên tìm kiếm (noindex).
  Công cụ chạy từ web Nhà Đất Mr. Tâm ở chế độ nhúng (?nhung=1): Viết 5 tin AI, Viết tin nhanh, tra quy hoạch.
  Không hiện tên, số điện thoại, Zalo của Mr. Tâm. Khóa AI nằm trên máy chủ AI riêng, không nằm trong file này.
  Muốn gỡ trang: xóa file này.
-->
<template>
  <main class="ai-tool-page">
    <div class="container">
      <nav class="ai-tool-breadcrumb">
        <NuxtLink to="/">Trang Chủ</NuxtLink>
        <i class="fa-solid fa-chevron-right"></i>
        <span class="active-crumb">Công Cụ Viết Tin AI</span>
      </nav>

      <iframe
        class="ai-tool-frame"
        src="https://nha-dat-mr-tam.pages.dev/?nhung=1"
        title="Công cụ viết tin AI"
        allow="clipboard-write"
      ></iframe>

      <p class="ai-tool-note">
        Lần đầu dùng: đăng nhập nhân viên ngay trong khung bên trên.
        Không hiện công cụ?
        <a href="https://nha-dat-mr-tam.pages.dev/?nhung=1" target="_blank" rel="noopener">Mở ở tab mới</a>
      </p>
    </div>
  </main>
</template>

<script setup lang="ts">
useHead({
  title: 'Công Cụ Viết Tin AI | Bến Thành Land',
  meta: [{ name: 'robots', content: 'noindex, nofollow' }]
});
</script>

<style scoped>
.ai-tool-page {
  padding: 100px 0 40px;
  min-height: 100vh;
}

.ai-tool-breadcrumb {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 0.85rem;
  color: var(--text-muted);
  margin-bottom: 16px;
}

.ai-tool-breadcrumb a {
  color: var(--gold-primary);
  text-decoration: none;
}

.ai-tool-breadcrumb .active-crumb {
  color: var(--text-main);
  font-weight: 600;
}

.ai-tool-frame {
  display: block;
  width: 100%;
  height: calc(100vh - 170px);
  min-height: 640px;
  border: 1px solid var(--border-color);
  border-radius: 12px;
  background: #fff;
}

.ai-tool-note {
  margin: 10px 0 0;
  text-align: center;
  font-size: 0.85rem;
  color: var(--text-muted);
}

.ai-tool-note a {
  color: var(--gold-primary);
  font-weight: 600;
}
</style>
