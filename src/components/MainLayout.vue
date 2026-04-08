<script setup lang="ts">
import { Pages, useGlobalState } from '@/composables/app-state';
import IconRustLang from './IconRustLang.vue';
import IconVueJs from './IconVueJs.vue';

// Layout component for consistent page structure

const appState = useGlobalState();
const { page, setPage } = appState;

</script>

<template>
  <div class="layout-shell flex flex-col h-dvh overflow-hidden">
    <header class="layout-header">
      <div class="container mx-auto px-4 py-4 md:px-8 flex items-center justify-between gap-4">
        <div class="flex items-center space-x-3">
          <img src="/logo.png" alt="Logo" class="h-9 w-9 rounded-xl p-1.5 bg-slate-900/60 border border-white/10" />
          <div>
            <h2 class="text-base md:text-lg font-semibold tracking-wide text-slate-100">Quest Dashboard</h2>
          </div>
        </div>
        <nav>
          <ul class="flex space-x-2 md:space-x-3">
            <li>
              <a href="#" 
                class="nav-link"
                :class="{ 'nav-link-active': page === Pages.HOME }"
                @click.prevent="setPage(Pages.HOME)"
              >
                Home
              </a>
            </li> 
            <li>
              <a href="#" 
                class="nav-link"
                :class="{ 'nav-link-active': page === Pages.PLAYGROUND }"
                @click.prevent="setPage(Pages.PLAYGROUND)"
              >
                Playground
              </a>
            </li>
          </ul>
        </nav>
      </div>
    </header>
    
    <main class="flex-grow overflow-y-auto main-scroll">
      <slot></slot>
    </main>
    
    <footer class="layout-footer mt-auto">
      <div class="container mx-auto px-4 py-4 text-center text-sm text-slate-400">
        &copy; 2025 Built with <IconRustLang class="inline-block h-4 w-4 text-red-200 dark:text-white mx-0.5"/> Rust and <IconVueJs class="h-4 w-4 inline-block text-[#4FC08D] mx-0.5"/> Vue.js
        <span class="mx-2 text-slate-600">|</span>
        Contributor:
        <a
          href="https://github.com/msxfury"
          target="_blank"
          rel="noopener noreferrer"
          class="ml-1 contributor-link"
        >
          msxfury
        </a>
      </div>
    </footer>
  </div>
</template>

<style scoped>
@reference "../theme/style.css";

.layout-shell {
  background: transparent;
}

.layout-header {
  border-bottom: 1px solid rgba(148, 163, 184, 0.18);
  background: linear-gradient(180deg, rgba(10, 17, 31, 0.92) 0%, rgba(11, 18, 32, 0.78) 100%);
  backdrop-filter: blur(10px);
  box-shadow: 0 6px 22px rgba(2, 8, 23, 0.35);
}

.nav-link {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border-radius: 999px;
  border: 1px solid transparent;
  color: #94a3b8;
  font-size: 0.875rem;
  font-weight: 500;
  padding: 0.45rem 0.95rem;
  transition: all var(--trans-fast);
}

.nav-link:hover {
  color: #e2e8f0;
  border-color: rgba(148, 163, 184, 0.25);
  background: rgba(15, 23, 42, 0.65);
}

.nav-link-active {
  color: #f5f3ff;
  border-color: rgba(124, 58, 237, 0.48);
  background: linear-gradient(135deg, rgba(124, 58, 237, 0.24), rgba(34, 211, 238, 0.22));
  box-shadow: 0 0 18px rgba(124, 58, 237, 0.35);
}

.nav-link-disabled {
  opacity: 0.75;
}

.main-scroll {
  scrollbar-width: thin;
  scrollbar-color: rgba(148, 163, 184, 0.5) transparent;
}

.layout-footer {
  border-top: 1px solid rgba(148, 163, 184, 0.18);
  background: linear-gradient(180deg, rgba(11, 18, 32, 0.68) 0%, rgba(7, 13, 23, 0.95) 100%);
}

.contributor-link {
  color: #a5b4fc;
  text-decoration: none;
  transition: color var(--trans-fast);
}

.contributor-link:hover {
  color: #22d3ee;
}
</style>
