<template>
  <nav class="main-menu">
    <NuxtLink
      v-for="item in menuItems"
      :key="item.to"
      :to="item.to"
      class="menu-link"
      :class="{ active: route.path === item.to }"
    >
      {{ item.label }}
    </NuxtLink>
  </nav>
</template>

<script setup>
import { NuxtLink } from '#components'

const route = useRoute()
const menuItems = [
  { label: 'コメント一覧', to: '/comments' },
  { label: 'ライブチャット', to: '/live-chat' },
  { label: 'コメント数 TOP10', to: '/top-videos' },
]
</script>

<style scoped>
.main-menu {
  max-width: 960px;
  margin: 0 auto;
  padding: 12px 16px 0;
  display: flex;
  gap: 8px;
  flex-wrap: wrap;
  justify-content: center;
}

.menu-link {
  padding: 8px 20px;
  border-radius: 999px;
  border: 1px solid rgba(148, 163, 184, 0.45);
  color: #f3f4f6;
  text-decoration: none;
  font-size: 0.9rem;
  background: rgba(15, 23, 42, 0.8);
  transition: border-color 0.2s ease, background 0.2s ease, color 0.2s ease;
}

.menu-link:hover {
  border-color: rgba(248, 113, 113, 0.8);
  color: #fff;
}

.menu-link.active {
  background: linear-gradient(135deg, #f97373, #fb923c);
  border-color: transparent;
  color: #111827;
  font-weight: 600;
  box-shadow: 0 12px 26px rgba(248, 113, 113, 0.35);
}
</style>
