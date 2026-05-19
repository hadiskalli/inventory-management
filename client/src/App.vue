<template>
  <div class="app" :class="{ 'sidebar-collapsed': sidebarCollapsed }">
    <aside class="sidebar">
      <div class="sidebar-header">
        <div class="sidebar-logo">
          <span class="sidebar-logo-mark">N</span>
          <span class="sidebar-app-name">{{ t('nav.companyName') }}</span>
        </div>
        <button class="sidebar-toggle" @click="toggleSidebar" :title="sidebarCollapsed ? 'Expand sidebar' : 'Collapse sidebar'">
          <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.75" stroke-linecap="round">
            <path v-if="!sidebarCollapsed" d="M10 3L6 8l4 5"/>
            <path v-else d="M6 3l4 5-4 5"/>
          </svg>
        </button>
      </div>

      <nav class="sidebar-nav">
        <router-link to="/" :class="{ active: $route.path === '/' }" :data-tooltip="t('nav.overview')">
          <span class="nav-icon">
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5">
              <rect x="1" y="1" width="6" height="6" rx="1"/>
              <rect x="9" y="1" width="6" height="6" rx="1"/>
              <rect x="1" y="9" width="6" height="6" rx="1"/>
              <rect x="9" y="9" width="6" height="6" rx="1"/>
            </svg>
          </span>
          <span class="nav-label">{{ t('nav.overview') }}</span>
        </router-link>

        <router-link to="/inventory" :class="{ active: $route.path === '/inventory' }" :data-tooltip="t('nav.inventory')">
          <span class="nav-icon">
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5">
              <path d="M2 5l6-3 6 3v6l-6 3-6-3V5z"/>
              <path d="M8 2v12M2 5l6 3 6-3"/>
            </svg>
          </span>
          <span class="nav-label">{{ t('nav.inventory') }}</span>
        </router-link>

        <router-link to="/orders" :class="{ active: $route.path === '/orders' }" :data-tooltip="t('nav.orders')">
          <span class="nav-icon">
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5">
              <rect x="1" y="3" width="14" height="10" rx="1"/>
              <path d="M1 7h14M5 3V1M11 3V1"/>
            </svg>
          </span>
          <span class="nav-label">{{ t('nav.orders') }}</span>
        </router-link>

        <router-link to="/spending" :class="{ active: $route.path === '/spending' }" :data-tooltip="t('nav.finance')">
          <span class="nav-icon">
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5">
              <path d="M1 12l4-4 3 3 4-5 3 2"/>
              <path d="M1 14h14"/>
            </svg>
          </span>
          <span class="nav-label">{{ t('nav.finance') }}</span>
        </router-link>

        <router-link to="/demand" :class="{ active: $route.path === '/demand' }" :data-tooltip="t('nav.demandForecast')">
          <span class="nav-icon">
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5">
              <path d="M1 11c2-4 4-2 6-6s3 2 8-2"/>
              <path d="M11 3l3 0 0 3"/>
            </svg>
          </span>
          <span class="nav-label">{{ t('nav.demandForecast') }}</span>
        </router-link>

        <router-link to="/reports" :class="{ active: $route.path === '/reports' }" data-tooltip="Reports">
          <span class="nav-icon">
            <svg viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.5">
              <rect x="2" y="1" width="12" height="14" rx="1"/>
              <path d="M5 5h6M5 8h6M5 11h4"/>
            </svg>
          </span>
          <span class="nav-label">Reports</span>
        </router-link>
      </nav>

      <div class="sidebar-footer">
        <LanguageSwitcher />
        <ProfileMenu
          @show-profile-details="showProfileDetails = true"
          @show-tasks="showTasks = true"
        />
      </div>
    </aside>

    <div class="main-wrapper">
      <FilterBar />
      <main class="main-content">
        <router-view />
      </main>
    </div>

    <ProfileDetailsModal
      :is-open="showProfileDetails"
      @close="showProfileDetails = false"
    />

    <TasksModal
      :is-open="showTasks"
      :tasks="tasks"
      @close="showTasks = false"
      @add-task="addTask"
      @delete-task="deleteTask"
      @toggle-task="toggleTask"
    />
  </div>
</template>

<script>
import { ref, onMounted, onUnmounted, computed } from 'vue'
import { api } from './api'
import { useAuth } from './composables/useAuth'
import { useI18n } from './composables/useI18n'
import FilterBar from './components/FilterBar.vue'
import ProfileMenu from './components/ProfileMenu.vue'
import ProfileDetailsModal from './components/ProfileDetailsModal.vue'
import TasksModal from './components/TasksModal.vue'
import LanguageSwitcher from './components/LanguageSwitcher.vue'

export default {
  name: 'App',
  components: {
    FilterBar,
    ProfileMenu,
    ProfileDetailsModal,
    TasksModal,
    LanguageSwitcher
  },
  setup() {
    const { currentUser } = useAuth()
    const { t } = useI18n()
    const showProfileDetails = ref(false)
    const showTasks = ref(false)
    const apiTasks = ref([])

    // Sidebar collapse — restore from localStorage, default to collapsed below 1024px
    const sidebarCollapsed = ref(
      localStorage.getItem('sidebarCollapsed') !== null
        ? localStorage.getItem('sidebarCollapsed') === 'true'
        : window.innerWidth < 1024
    )

    const toggleSidebar = () => {
      sidebarCollapsed.value = !sidebarCollapsed.value
      localStorage.setItem('sidebarCollapsed', String(sidebarCollapsed.value))
    }

    // Auto-collapse when viewport shrinks below 1024px, auto-expand above
    const handleResize = () => {
      const small = window.innerWidth < 1024
      // Only override if no explicit user preference is stored
      if (localStorage.getItem('sidebarCollapsed') === null) {
        sidebarCollapsed.value = small
      }
    }

    onUnmounted(() => {
      window.removeEventListener('resize', handleResize)
    })

    // Merge mock tasks from currentUser with API tasks
    const tasks = computed(() => {
      return [...currentUser.value.tasks, ...apiTasks.value]
    })

    const addTask = (taskData) => {
      apiTasks.value.unshift({
        id: `task-${Date.now()}`,
        status: 'pending',
        ...taskData
      })
    }

    const deleteTask = (taskId) => {
      const isMockTask = currentUser.value.tasks.some(t => t.id === taskId)
      if (isMockTask) {
        const index = currentUser.value.tasks.findIndex(t => t.id === taskId)
        if (index !== -1) {
          currentUser.value.tasks.splice(index, 1)
        }
      } else {
        apiTasks.value = apiTasks.value.filter(t => t.id !== taskId)
      }
    }

    const toggleTask = (taskId) => {
      const mockTask = currentUser.value.tasks.find(t => t.id === taskId)
      if (mockTask) {
        mockTask.status = mockTask.status === 'pending' ? 'completed' : 'pending'
      } else {
        const task = apiTasks.value.find(t => t.id === taskId)
        if (task) {
          task.status = task.status === 'pending' ? 'completed' : 'pending'
        }
      }
    }

    onMounted(() => {
      window.addEventListener('resize', handleResize)
    })

    return {
      t,
      showProfileDetails,
      showTasks,
      tasks,
      addTask,
      deleteTask,
      toggleTask,
      sidebarCollapsed,
      toggleSidebar
    }
  }
}
</script>

<style>
/* ── Reset ── */
*, *::before, *::after {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}

/* ── Design Tokens ── */
:root {
  --sidebar-bg: #0f172a;
  --sidebar-width: 260px;
  --sidebar-text: #94a3b8;
  --sidebar-text-active: #f1f5f9;
  --sidebar-text-hover: #cbd5e1;
  --sidebar-accent: #3b82f6;
  --sidebar-hover: rgba(255, 255, 255, 0.06);
  --sidebar-border: rgba(255, 255, 255, 0.08);

  --surface: #ffffff;
  --surface-raised: #f8fafc;
  --surface-sunken: #f1f5f9;

  --border: #e2e8f0;
  --border-strong: #cbd5e1;

  --text-primary: #0f172a;
  --text-secondary: #334155;
  --text-muted: #64748b;
  --text-faint: #94a3b8;

  --color-success: #059669;
  --color-warning: #d97706;
  --color-danger: #dc2626;
  --color-info: #2563eb;
  --color-success-bg: #d1fae5;
  --color-warning-bg: #fef3c7;
  --color-danger-bg: #fee2e2;
  --color-info-bg: #dbeafe;

  --space-1: 4px;   --space-2: 8px;   --space-3: 12px;
  --space-4: 16px;  --space-5: 20px;  --space-6: 24px;
  --space-8: 32px;  --space-10: 40px;

  --radius-sm: 6px;
  --radius-md: 8px;
  --radius-lg: 12px;
}

/* ── Base ── */
body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  background: var(--surface-raised);
  color: var(--text-primary);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

/* ── App Shell ── */
.app {
  display: flex;
  min-height: 100vh;
}

/* ── Sidebar ── */
.sidebar {
  width: var(--sidebar-width);
  min-height: 100vh;
  background: var(--sidebar-bg);
  display: flex;
  flex-direction: column;
  position: fixed;
  top: 0; left: 0; bottom: 0;
  overflow-y: auto;
  overflow-x: hidden;
  z-index: 50;
  scrollbar-width: none;
  transition: width 0.22s ease;
}
.sidebar::-webkit-scrollbar { display: none; }

.sidebar-header {
  padding: var(--space-5) var(--space-4);
  border-bottom: 1px solid var(--sidebar-border);
  flex-shrink: 0;
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: var(--space-2);
  min-height: 64px;
}

.sidebar-logo {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  overflow: hidden;
  min-width: 0;
}

.sidebar-logo-mark {
  width: 32px;
  height: 32px;
  background: var(--sidebar-accent);
  border-radius: var(--radius-sm);
  display: grid;
  place-items: center;
  font-weight: 700;
  font-size: 0.875rem;
  color: white;
  flex-shrink: 0;
}

.sidebar-app-name {
  font-size: 0.9375rem;
  font-weight: 600;
  color: var(--sidebar-text-active);
  letter-spacing: -0.01em;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.sidebar-nav {
  padding: var(--space-4) var(--space-3);
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.sidebar-nav a {
  display: flex;
  align-items: center;
  gap: var(--space-3);
  padding: 9px var(--space-3);
  color: var(--sidebar-text);
  text-decoration: none;
  font-size: 0.875rem;
  font-weight: 500;
  border-radius: var(--radius-sm);
  border-left: 3px solid transparent;
  transition: background 0.15s ease, color 0.15s ease, border-color 0.15s ease;
}

.sidebar-nav a:hover {
  background: var(--sidebar-hover);
  color: var(--sidebar-text-hover);
}

.sidebar-nav a.active {
  background: var(--sidebar-hover);
  color: var(--sidebar-text-active);
  border-left-color: var(--sidebar-accent);
}

.nav-icon {
  width: 16px;
  height: 16px;
  flex-shrink: 0;
  opacity: 0.65;
}

.sidebar-nav a.active .nav-icon,
.sidebar-nav a:hover .nav-icon { opacity: 1; }

.sidebar-footer {
  padding: var(--space-4) var(--space-3);
  border-top: 1px solid var(--sidebar-border);
  margin-top: auto;
  display: flex;
  flex-direction: column;
  gap: var(--space-2);
  flex-shrink: 0;
}

/* ── Toggle Button ── */
.sidebar-toggle {
  width: 28px;
  height: 28px;
  display: flex;
  align-items: center;
  justify-content: center;
  background: transparent;
  border: 1px solid var(--sidebar-border);
  border-radius: var(--radius-sm);
  color: var(--sidebar-text);
  cursor: pointer;
  flex-shrink: 0;
  transition: background 0.15s ease, color 0.15s ease, border-color 0.15s ease;
}
.sidebar-toggle:hover {
  background: var(--sidebar-hover);
  color: var(--sidebar-text-active);
  border-color: rgba(255, 255, 255, 0.15);
}
.sidebar-toggle svg {
  width: 14px;
  height: 14px;
}

/* ── Main Area ── */
.main-wrapper {
  flex: 1;
  margin-left: var(--sidebar-width);
  min-height: 100vh;
  display: flex;
  flex-direction: column;
  min-width: 0;
  transition: margin-left 0.22s ease;
}

.main-content {
  flex: 1;
  padding: var(--space-8);
}

/* ── Page Header ── */
.page-header {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  margin-bottom: var(--space-8);
}

.page-header h2 {
  font-size: 1.5rem;
  font-weight: 700;
  color: var(--text-primary);
  letter-spacing: -0.025em;
  margin-bottom: var(--space-1);
}

.page-header p {
  font-size: 0.875rem;
  color: var(--text-muted);
}

/* ── Stats Grid ── */
.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
  gap: var(--space-5);
  margin-bottom: var(--space-8);
}

.stat-card {
  background: var(--surface);
  padding: var(--space-6);
  border-radius: var(--radius-lg);
  border: 1px solid var(--border);
  transition: box-shadow 0.15s ease;
}

.stat-card:hover {
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.06);
}

.stat-label {
  color: var(--text-muted);
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  margin-bottom: var(--space-2);
  display: block;
}

.stat-value {
  font-size: 2rem;
  font-weight: 700;
  color: var(--text-primary);
  letter-spacing: -0.025em;
  line-height: 1;
  display: block;
}

.stat-card.warning .stat-value { color: var(--color-warning); }
.stat-card.success .stat-value { color: var(--color-success); }
.stat-card.danger  .stat-value { color: var(--color-danger); }
.stat-card.info    .stat-value { color: var(--color-info); }

/* ── Card ── */
.card {
  background: var(--surface);
  border-radius: var(--radius-lg);
  border: 1px solid var(--border);
  margin-bottom: var(--space-6);
  overflow: hidden;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: var(--space-5) var(--space-6);
  border-bottom: 1px solid var(--border);
}

.card-title {
  font-size: 0.9375rem;
  font-weight: 600;
  color: var(--text-primary);
  letter-spacing: -0.01em;
}

/* ── Tables ── */
.table-container { overflow-x: auto; }

table { width: 100%; border-collapse: collapse; }

thead { background: var(--surface-raised); }

th {
  text-align: left;
  padding: var(--space-3) var(--space-4);
  font-weight: 600;
  color: var(--text-muted);
  font-size: 0.6875rem;
  text-transform: uppercase;
  letter-spacing: 0.07em;
  border-bottom: 1px solid var(--border);
  white-space: nowrap;
}

td {
  padding: var(--space-3) var(--space-4);
  border-bottom: 1px solid var(--surface-raised);
  color: var(--text-secondary);
  font-size: 0.875rem;
}

tbody tr:last-child td { border-bottom: none; }
tbody tr { transition: background-color 0.1s ease; }
tbody tr:hover td { background: var(--surface-raised); }

/* ── Badges ── */
.badge {
  display: inline-flex;
  align-items: center;
  padding: 3px 10px;
  border-radius: 999px;
  font-size: 0.6875rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  white-space: nowrap;
}

.badge.success    { background: var(--color-success-bg); color: var(--color-success); }
.badge.warning    { background: var(--color-warning-bg); color: var(--color-warning); }
.badge.danger     { background: var(--color-danger-bg);  color: var(--color-danger); }
.badge.info       { background: var(--color-info-bg);    color: var(--color-info); }
.badge.increasing { background: var(--color-success-bg); color: var(--color-success); }
.badge.decreasing { background: var(--color-danger-bg);  color: var(--color-danger); }
.badge.stable     { background: var(--color-info-bg);    color: var(--color-info); }
.badge.high       { background: var(--color-danger-bg);  color: var(--color-danger); }
.badge.medium     { background: var(--color-warning-bg); color: var(--color-warning); }
.badge.low        { background: var(--color-info-bg);    color: var(--color-info); }

/* ── States ── */
.loading {
  text-align: center;
  padding: 3rem;
  color: var(--text-muted);
  font-size: 0.9375rem;
}

.error {
  background: var(--color-danger-bg);
  border: 1px solid #fca5a5;
  color: var(--color-danger);
  padding: var(--space-4);
  border-radius: var(--radius-md);
  margin: var(--space-4) 0;
  font-size: 0.875rem;
}

/* ── Collapsed Sidebar ── */
.sidebar-collapsed .sidebar {
  width: 64px;
}

.sidebar-collapsed .main-wrapper {
  margin-left: 64px;
}

/* Hide text labels */
.sidebar-collapsed .sidebar-app-name,
.sidebar-collapsed .nav-label {
  display: none;
}

/* Center header contents */
.sidebar-collapsed .sidebar-header {
  justify-content: center;
  flex-direction: column-reverse;
  gap: var(--space-2);
  padding: var(--space-3) var(--space-2);
}

.sidebar-collapsed .sidebar-logo {
  justify-content: center;
}

/* Center nav icons */
.sidebar-collapsed .sidebar-nav {
  padding: var(--space-4) var(--space-2);
}

.sidebar-collapsed .sidebar-nav a {
  justify-content: center;
  padding: 10px;
  border-left: 3px solid transparent;
  /* Keep active border visible via a bottom border on collapsed */
}

.sidebar-collapsed .sidebar-nav a .nav-icon {
  width: 18px;
  height: 18px;
}

/* Tooltip on hover */
.sidebar-collapsed .sidebar-nav a {
  position: relative;
}

.sidebar-collapsed .sidebar-nav a::after {
  content: attr(data-tooltip);
  position: absolute;
  left: calc(100% + 10px);
  top: 50%;
  transform: translateY(-50%);
  background: #1e293b;
  color: #f1f5f9;
  padding: 5px 10px;
  border-radius: var(--radius-sm);
  font-size: 0.8125rem;
  font-weight: 500;
  white-space: nowrap;
  opacity: 0;
  pointer-events: none;
  transition: opacity 0.12s ease;
  z-index: 200;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.25);
}

.sidebar-collapsed .sidebar-nav a:hover::after {
  opacity: 1;
}

/* Footer in collapsed state */
.sidebar-collapsed .sidebar-footer {
  align-items: center;
  padding: var(--space-3) var(--space-2);
}

/* Slim down profile and language buttons in collapsed state */
.sidebar-collapsed .profile-button {
  justify-content: center;
  padding: 8px;
  width: auto;
}

.sidebar-collapsed .profile-name,
.sidebar-collapsed .profile-button .chevron {
  display: none;
}

.sidebar-collapsed .language-button {
  justify-content: center;
  padding: 8px;
  width: auto;
}

.sidebar-collapsed .language-label,
.sidebar-collapsed .language-button .chevron {
  display: none;
}
</style>
