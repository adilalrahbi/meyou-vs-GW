<template>
  <div class="projects-view">
    <!-- Header -->
    <div class="page-hdr">
      <div class="page-title">جميع المشاريع</div>
      <div class="hdr-btns">
        <RouterLink to="/report" class="btn-rep">📊 تقرير</RouterLink>
        <select v-model="selectedYear" class="year-sel" @change="applyFilters">
          <option value="all">📅 كل السنوات</option>
          <option v-for="year in availableYears" :key="year" :value="year">
            {{ year }}
          </option>
        </select>
        <RouterLink v-if="authStore.isEditor" to="/form" class="btn-new">
          + إضافة مشروع
        </RouterLink>
      </div>
    </div>

    <!-- Filters -->
    <div class="filter-bar">
      <button
        v-for="f in filters"
        :key="f.value"
        class="fpill"
        :class="{ on: selectedFilter === f.value }"
        @click="selectedFilter = f.value"
      >
        {{ f.label }}
      </button>
    </div>

    <!-- Summary Stats -->
    <div class="type-counts">
      <div class="tc-box" style="background: var(--blue-bg); border-color: #c5d3ed">
        <span class="badge b-c">تجاري</span>
        <span class="tc-n" style="color: var(--blue)">{{ metrics.commercialProjects }}</span>
      </div>
      <div class="tc-box" style="background: var(--amber-bg); border-color: #f0c870">
        <span class="badge b-i">داخلي</span>
        <span class="tc-n" style="color: var(--amber)">{{ metrics.internalProjects }}</span>
      </div>
      <div class="tc-box" style="background: #edf5ff; border-color: #5b9bd5">
        <span class="badge b-v">استثماري</span>
        <span class="tc-n" style="color: #1a5fa8">{{ metrics.investmentProjects }}</span>
      </div>
      <div class="tc-box" style="background: #f5f5f5; border-color: #ddd">
        <span style="font-size: 11px; font-weight: 600; color: var(--text3); text-transform: uppercase"
          >الإجمالي</span
        >
        <span class="tc-n">{{ metrics.totalProjects }}</span>
      </div>
    </div>

    <!-- Summary Strip -->
    <div class="sum-strip">
      <div class="sbox" style="background: var(--blue-bg); border-color: #c5d3ed">
        <div class="sl" style="color: var(--blue)">مشاريع تجارية</div>
        <div class="sv" style="color: var(--blue)">{{ metrics.commercialProjects }}</div>
      </div>
      <div class="sbox" style="background: var(--blue-bg); border-color: #c5d3ed">
        <div class="sl" style="color: var(--blue)">قيمة التجاري</div>
        <div class="sv" style="color: var(--blue)">{{ formatCurrency(metrics.totalDealValue) }}</div>
      </div>
      <div class="sbox" style="background: var(--red-bg); border-color: #f2c4bf">
        <div class="sl" style="color: var(--red)">صرف MeYou</div>
        <div class="sv" style="color: var(--red)">{{ formatCurrency(metrics.totalCosts) }}</div>
      </div>
      <div class="sbox" style="background: var(--pink-bg); border-color: #f0c8cf">
        <div class="sl" style="color: #9b3f50">مستحق MeYou</div>
        <div class="sv" style="color: #9b3f50">{{ formatCurrency(projectStore.totalDue) }}</div>
      </div>
      <div class="sbox" style="background: var(--green-bg); border-color: #b2dfc8">
        <div class="sl" style="color: var(--green)">الأرباح</div>
        <div class="sv" style="color: var(--green)">{{ formatCurrency(metrics.totalProfit) }}</div>
      </div>
      <div class="sbox" style="background: var(--green-bg); border-color: #b2dfc8">
        <div class="sl" style="color: var(--green)">ربح MeYou 50%</div>
        <div class="sv" style="color: var(--green)">{{ formatCurrency(metrics.meyouProfit) }}</div>
      </div>
      <div class="sbox" style="background: #1c1c1c; border-color: #333">
        <div class="sl" style="color: #999">ربح GW 50%</div>
        <div class="sv" style="color: #fff">{{ formatCurrency(metrics.gwProfit) }}</div>
      </div>

      <!-- Remaining Banner -->
      <div class="sbox rem-banner" :style="remainingBannerStyle" style="grid-column: span 7">
        <div class="rem-left">
          <span class="rem-icon" :class="{ pulse: remainingAmount > 0 }">
            {{ remainingIcon }}
          </span>
          <div>
            <div class="rem-title">{{ remainingTitle }}</div>
            <div class="rem-sub">{{ remainingSubtitle }}</div>
          </div>
        </div>
        <div class="rem-val">{{ formatCurrency(remainingAmount) }}</div>
      </div>
    </div>

    <!-- Projects List -->
    <div v-if="filteredProjects.length === 0" class="empty">
      <div class="ei">📂</div>
      <p>لا توجد مشاريع</p>
    </div>
    <div v-else>
      <ProjectCard
        v-for="project in filteredProjects"
        :key="project.id"
        :project="project"
        :allocation="allocations[project.id]"
      />
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'
import { RouterLink } from 'vue-router'
import { useProjectStore } from '../stores/projectStore'
import { useAuthStore } from '../stores/authStore'
import { calculateProjectMetrics, formatCurrency, fifoAllocation } from '../utils/calculations'
import ProjectCard from '../components/ProjectCard.vue'
import type { Project } from '../types'

const projectStore = useProjectStore()
const authStore = useAuthStore()

// Data
const selectedYear = ref('all')
const selectedFilter = ref('all')

const filters = [
  { value: 'all', label: 'الكل' },
  { value: 'progress', label: 'جاري' },
  { value: 'done', label: 'مكتمل' },
  { value: 'unpaid', label: 'لم يُدفع' },
  { value: 'partial', label: 'جزئي' },
  { value: 'paid', label: 'مدفوع' },
]

// Computed
const availableYears = computed(() => {
  const years = new Set(projectStore.projects.map(p => p.date.substring(0, 4)))
  return Array.from(years).sort().reverse()
})

const yearFilteredProjects = computed(() => {
  if (selectedYear.value === 'all') return projectStore.projects
  return projectStore.projects.filter(p => p.date.startsWith(selectedYear.value))
})

const { allocation: allocations } = computed(() => fifoAllocation(yearFilteredProjects.value, projectStore.galaxyWayPayments)).value

const metrics = computed(() => calculateProjectMetrics(yearFilteredProjects.value))

const filteredProjects = computed(() => {
  return yearFilteredProjects.value.filter(p => {
    if (selectedFilter.value === 'all') return true
    if (selectedFilter.value === 'progress') return p.projStatus === 'progress'
    if (selectedFilter.value === 'done') return p.projStatus === 'done'
    return p.payStatus === selectedFilter.value
  }).reverse()
})

const remainingAmount = computed(() => projectStore.totalRemaining)

const remainingBannerStyle = computed(() => {
  if (projectStore.totalRemaining > 0.001) {
    return {
      background: 'linear-gradient(135deg, #c0392b, #96281b)',
      borderColor: '#c0392b',
    }
  } else if (projectStore.totalRemaining < 0.001) {
    return {
      background: 'linear-gradient(135deg, #0a7c4e, #076640)',
      borderColor: '#0a7c4e',
    }
  }
  return {}
})

const remainingIcon = computed(() => {
  if (projectStore.totalRemaining < 0.001) return '🟢'
  return '🔴'
})

const remainingTitle = computed(() => {
  const over = projectStore.totalPaid - projectStore.totalDue
  if (over > 0.001) return '⚠️ دُفع أكثر من المستحق'
  if (projectStore.totalRemaining < 0.001) return 'المتبقي لدفعه لـ MeYou'
  return 'المتبقي لدفعه لـ MeYou'
})

const remainingSubtitle = computed(() => {
  const over = projectStore.totalPaid - projectStore.totalDue
  if (over > 0.001) return 'رصيد زائد على الدفعات القادمة'
  if (projectStore.totalRemaining < 0.001) return 'من المشاريع المكتملة'
  return 'من المشاريع المكتملة'
})

// Methods
const applyFilters = () => {
  // Filters applied automatically via computed properties
}

// Lifecycle
onMounted(() => {
  projectStore.loadData()
})
</script>

<style scoped>
.projects-view {
  animation: fadeIn 0.3s ease-in;
}

@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(10px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.page-hdr {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 16px;
  flex-wrap: wrap;
  gap: 8px;
}

.page-title {
  font-size: 17px;
  font-weight: 600;
}

.hdr-btns {
  display: flex;
  align-items: center;
  gap: 8px;
  flex-wrap: wrap;
}

.btn-new,
.btn-rep {
  padding: 8px 14px;
  border-radius: 8px;
  font-size: 12px;
  font-weight: 600;
  cursor: pointer;
  text-decoration: none;
  transition: all 0.2s;
}

.btn-new {
  background: #233d79;
  color: #fff;
  border: none;
}

.btn-new:hover {
  background: #3455a0;
}

.btn-rep {
  background: #f0ebfc;
  border: 1px solid #d0bef5;
  color: #5b35b0;
}

.btn-rep:hover {
  background: #e0d5f5;
}

.year-sel {
  font-size: 12px;
  padding: 7px 10px;
  border: 1px solid #ddd;
  border-radius: 8px;
  background: #fafafa;
  outline: none;
  cursor: pointer;
}

.filter-bar {
  display: flex;
  gap: 6px;
  margin-bottom: 14px;
  flex-wrap: wrap;
}

.fpill {
  padding: 5px 13px;
  border: 1px solid #ddd;
  border-radius: 20px;
  background: #fafafa;
  font-size: 11px;
  font-weight: 500;
  cursor: pointer;
  color: #555;
  transition: all 0.2s;
}

.fpill:hover {
  border-color: #233d79;
  color: #233d79;
}

.fpill.on {
  background: #233d79;
  border-color: #233d79;
  color: #fff;
}

.type-counts {
  display: flex;
  gap: 8px;
  margin-bottom: 12px;
  flex-wrap: wrap;
}

.tc-box {
  border-radius: 8px;
  padding: 8px 14px;
  display: flex;
  align-items: center;
  gap: 8px;
  border: 1px solid transparent;
}

.tc-n {
  font-family: 'IBM Plex Mono', monospace;
  font-size: 15px;
  font-weight: 600;
}

.sum-strip {
  display: grid;
  grid-template-columns: repeat(7, 1fr);
  gap: 8px;
  margin-bottom: 14px;
}

.sbox {
  background: #fff;
  border: 1px solid #ddd;
  border-radius: 8px;
  padding: 12px 13px;
}

.sl {
  font-size: 9px;
  font-weight: 600;
  letter-spacing: 0.05em;
  text-transform: uppercase;
  margin-bottom: 4px;
}

.sv {
  font-family: 'IBM Plex Mono', monospace;
  font-size: 13px;
  font-weight: 500;
}

.rem-banner {
  grid-column: span 7;
  border-width: 2px;
  display: flex;
  align-items: center;
  justify-content: space-between;
  position: relative;
  overflow: hidden;
}

.rem-banner::before {
  content: '';
  position: absolute;
  inset: 0;
  background: repeating-linear-gradient(
    45deg,
    transparent,
    transparent 8px,
    rgba(255, 255, 255, 0.03) 8px,
    rgba(255, 255, 255, 0.03) 16px
  );
  pointer-events: none;
}

.rem-left {
  display: flex;
  align-items: center;
  gap: 10px;
  position: relative;
  z-index: 1;
}

.rem-icon {
  font-size: 20px;
}

.rem-icon.pulse {
  animation: pulse 1.5s infinite;
}

@keyframes pulse {
  0%,
  100% {
    opacity: 1;
  }
  50% {
    opacity: 0.3;
  }
}

.rem-title {
  font-size: 11px;
  font-weight: 700;
  color: rgba(255, 255, 255, 0.8);
  text-transform: uppercase;
  letter-spacing: 0.06em;
  margin-bottom: 2px;
}

.rem-sub {
  font-size: 10px;
  color: rgba(255, 255, 255, 0.5);
}

.rem-val {
  font-family: 'IBM Plex Mono', monospace;
  font-size: 24px;
  font-weight: 700;
  color: #fff;
  position: relative;
  z-index: 1;
}

.empty {
  text-align: center;
  padding: 50px 20px;
  color: #888;
}

.ei {
  font-size: 36px;
  margin-bottom: 10px;
  opacity: 0.4;
}

@media (max-width: 680px) {
  .sum-strip {
    grid-template-columns: repeat(2, 1fr);
  }

  .rem-banner {
    grid-column: span 2;
  }

  .page-hdr {
    flex-direction: column;
    align-items: flex-start;
  }

  .hdr-btns {
    width: 100%;
  }
}
</style>
