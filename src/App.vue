<template>
  <div class="app">
    <header class="header">
      <div class="header-content">
        <div class="title-wrapper">
          <div class="title-text">
            <h1 class="title">W-Track</h1>
            <p class="subtitle">Seguimiento de Peso</p>
          </div>
        </div>
      </div>
    </header>

    <main class="main-content">
      <!-- Peso Actual -->
      <section class="current-weight-card">
        <div class="weight-display">
          <p class="weight-label">Peso Actual</p>
          <p class="weight-value">{{ currentWeight || '--' }} <span class="weight-unit">kg</span></p>
          <p v-if="lastUpdate" class="last-update">Última actualización: {{ formatDate(lastUpdate) }}</p>
        </div>
      </section>

      <!-- Formulario para agregar peso -->
      <section class="form-card">
        <h2 class="section-title">Registrar Peso</h2>
        <form @submit.prevent="addWeight" class="weight-form">
          <div class="input-group">
            <label for="weight">Peso (kg)</label>
            <input
              id="weight"
              v-model.number="newWeight"
              type="number"
              step="0.1"
              min="0"
              max="500"
              placeholder="Ej: 75.5"
              required
              class="weight-input"
            />
          </div>
          <button type="submit" class="btn-primary">Guardar Peso</button>
        </form>
      </section>

      <!-- Gráfico -->
      <section v-if="weightHistory.length > 0" class="chart-card">
        <h2 class="section-title">Evolución</h2>
        <div class="chart-container">
          <WeightChart :data="chartData" />
        </div>
      </section>

      <!-- Historial -->
      <section v-if="weightHistory.length > 0" class="history-card">
        <h2 class="section-title">Historial</h2>
        <div class="history-list">
          <div
            v-for="(entry, index) in sortedHistory"
            :key="index"
            class="history-item-wrapper"
            @touchstart="handleTouchStart($event, index)"
            @touchmove="handleTouchMove($event, index)"
            @touchend="handleTouchEnd($event, index)"
            @mousedown="handleMouseDown($event, index)"
            @mousemove="handleMouseMove($event, index)"
            @mouseup="handleMouseUp($event, index)"
            @mouseleave="handleMouseLeave($event, index)"
          >
            <div
              class="history-item"
              :style="{ transform: `translateX(${swipeOffsets[index] || 0}px)` }"
            >
              <div class="history-content">
                <div class="history-date">
                  <span class="date-day">{{ formatDay(entry.date) }}</span>
                  <span class="date-month">{{ formatMonth(entry.date) }}</span>
                </div>
                <div class="history-weight">
                  <span class="weight-number">{{ entry.weight }}</span>
                  <span class="weight-unit-small">kg</span>
                </div>
                <div v-if="index > 0" class="history-diff">
                  <span
                    :class="[
                      'diff-badge',
                      entry.diff > 0 ? 'diff-positive' : entry.diff < 0 ? 'diff-negative' : 'diff-neutral'
                    ]"
                  >
                    {{ formatDiff(entry.diff) }}
                  </span>
                </div>
              </div>
            </div>
            <div class="delete-action" :class="{ 'delete-action-visible': (swipeOffsets[index] || 0) < -50 }">
              <span class="delete-text">Eliminar</span>
            </div>
          </div>
        </div>
      </section>

      <!-- Mensaje cuando no hay datos -->
      <section v-if="weightHistory.length === 0" class="empty-state">
        <div class="empty-icon">⚖️</div>
        <p class="empty-text">No hay registros aún</p>
        <p class="empty-subtext">Agrega tu primer peso para comenzar</p>
      </section>
    </main>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import WeightChart from './components/WeightChart.vue'

export default {
  name: 'App',
  components: {
    WeightChart
  },
  setup() {
    const newWeight = ref('')
    const weightHistory = ref([])
    const swipeOffsets = ref({})
    const touchStartX = ref({})
    const touchStartY = ref({})
    const isDragging = ref({})
    const mouseDownX = ref({})
    const isMouseDown = ref({})

    const currentWeight = computed(() => {
      if (weightHistory.value.length === 0) return null
      return weightHistory.value[weightHistory.value.length - 1].weight
    })

    const lastUpdate = computed(() => {
      if (weightHistory.value.length === 0) return null
      return weightHistory.value[weightHistory.value.length - 1].date
    })

    const sortedHistory = computed(() => {
      return [...weightHistory.value]
        .sort((a, b) => new Date(b.date) - new Date(a.date))
        .map((entry, index, arr) => {
          if (index < arr.length - 1) {
            const prevWeight = arr[index + 1].weight
            entry.diff = entry.weight - prevWeight
          } else {
            entry.diff = 0
          }
          return entry
        })
    })

    const chartData = computed(() => {
      const sorted = [...weightHistory.value].sort((a, b) => new Date(a.date) - new Date(b.date))
      return {
        labels: sorted.map(entry => formatChartDate(entry.date)),
        weights: sorted.map(entry => entry.weight)
      }
    })

    const loadData = () => {
      const saved = localStorage.getItem('weightHistory')
      if (saved) {
        try {
          weightHistory.value = JSON.parse(saved).map(entry => ({
            ...entry,
            date: new Date(entry.date)
          }))
        } catch (e) {
          console.error('Error loading data:', e)
          weightHistory.value = []
        }
      }
    }

    const saveData = () => {
      localStorage.setItem('weightHistory', JSON.stringify(weightHistory.value))
    }

    const addWeight = () => {
      if (!newWeight.value || newWeight.value <= 0) return

      const entry = {
        weight: parseFloat(newWeight.value),
        date: new Date()
      }

      weightHistory.value.push(entry)
      saveData()
      newWeight.value = ''
    }

    const deleteEntry = (index) => {
      const sorted = [...weightHistory.value].sort((a, b) => new Date(b.date) - new Date(a.date))
      const entryToDelete = sorted[index]
      weightHistory.value = weightHistory.value.filter(
        entry => entry.date.getTime() !== entryToDelete.date.getTime() || entry.weight !== entryToDelete.weight
      )
      saveData()
      // Reset swipe offset
      swipeOffsets.value[index] = 0
      delete swipeOffsets.value[index]
    }

    // Touch handlers
    const handleTouchStart = (e, index) => {
      touchStartX.value[index] = e.touches[0].clientX
      touchStartY.value[index] = e.touches[0].clientY
      isDragging.value[index] = false
    }

    const handleTouchMove = (e, index) => {
      if (touchStartX.value[index] === null || touchStartX.value[index] === undefined) return
      
      const currentX = e.touches[0].clientX
      const currentY = e.touches[0].clientY
      const diffX = currentX - touchStartX.value[index]
      const diffY = currentY - touchStartY.value[index]

      // Detect if horizontal swipe (more horizontal than vertical)
      if (!isDragging.value[index] && Math.abs(diffX) > Math.abs(diffY) && Math.abs(diffX) > 15) {
        isDragging.value[index] = true
      }

      if (isDragging.value[index] || Math.abs(diffX) > 10) {
        // Only allow swiping left (negative values)
        const offset = Math.min(0, diffX)
        swipeOffsets.value[index] = offset
        if (isDragging.value[index]) {
          e.preventDefault()
        }
      }
    }

    const handleTouchEnd = (e, index) => {
      if (touchStartX.value[index] === null || touchStartX.value[index] === undefined) return

      const offset = swipeOffsets.value[index] || 0
      const threshold = -80 // Swipe threshold to delete (reduced for easier deletion)

      if (offset < threshold && isDragging.value[index]) {
        // Delete the item
        deleteEntry(index)
      } else {
        // Snap back
        swipeOffsets.value[index] = 0
      }

      touchStartX.value[index] = null
      touchStartY.value[index] = null
      isDragging.value[index] = false
    }

    // Mouse handlers for desktop
    const handleMouseDown = (e, index) => {
      mouseDownX.value[index] = e.clientX
      isMouseDown.value[index] = true
    }

    const handleMouseMove = (e, index) => {
      if (!isMouseDown.value[index] || mouseDownX.value[index] === null) return

      const diffX = e.clientX - mouseDownX.value[index]
      const offset = Math.min(0, diffX)
      swipeOffsets.value[index] = offset
    }

    const handleMouseUp = (e, index) => {
      if (!isMouseDown.value[index]) return

      const offset = swipeOffsets.value[index] || 0
      const threshold = -80

      if (offset < threshold) {
        deleteEntry(index)
      } else {
        swipeOffsets.value[index] = 0
      }

      mouseDownX.value[index] = null
      isMouseDown.value[index] = false
    }

    const handleMouseLeave = (e, index) => {
      if (isMouseDown.value[index]) {
        const offset = swipeOffsets.value[index] || 0
        if (offset >= -80) {
          swipeOffsets.value[index] = 0
        }
        mouseDownX.value[index] = null
        isMouseDown.value[index] = false
      }
    }

    const formatDate = (date) => {
      if (!date) return ''
      const d = new Date(date)
      return d.toLocaleDateString('es-ES', {
        day: 'numeric',
        month: 'long',
        year: 'numeric',
        hour: '2-digit',
        minute: '2-digit'
      })
    }

    const formatDay = (date) => {
      const d = new Date(date)
      return d.getDate()
    }

    const formatMonth = (date) => {
      const d = new Date(date)
      return d.toLocaleDateString('es-ES', { month: 'short' })
    }

    const formatChartDate = (date) => {
      const d = new Date(date)
      return d.toLocaleDateString('es-ES', { day: 'numeric', month: 'short' })
    }

    const formatDiff = (diff) => {
      if (diff === 0) return '—'
      const sign = diff > 0 ? '+' : ''
      return `${sign}${diff.toFixed(1)}`
    }

    onMounted(() => {
      loadData()
    })

    return {
      newWeight,
      weightHistory,
      currentWeight,
      lastUpdate,
      sortedHistory,
      chartData,
      swipeOffsets,
      addWeight,
      deleteEntry,
      handleTouchStart,
      handleTouchMove,
      handleTouchEnd,
      handleMouseDown,
      handleMouseMove,
      handleMouseUp,
      handleMouseLeave,
      formatDate,
      formatDay,
      formatMonth,
      formatDiff
    }
  }
}
</script>

<style scoped>
.app {
  min-height: 100vh;
  padding-bottom: 2rem;
}

.header {
  background: var(--surface);
  border-bottom: 1px solid var(--border);
  padding: 1.5rem 1.5rem 2rem;
  position: relative;
  overflow: hidden;
}

.header::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 3px;
  background: linear-gradient(90deg, var(--primary) 0%, var(--secondary) 50%, var(--accent) 100%);
}

.header-content {
  max-width: 600px;
  margin: 0 auto;
}

.title-wrapper {
  display: flex;
  align-items: center;
}

.title-text {
  flex: 1;
}

.title {
  font-size: 2rem;
  font-weight: 800;
  margin: 0;
  letter-spacing: -1px;
  background: linear-gradient(135deg, var(--primary-light) 0%, var(--primary) 50%, var(--secondary) 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  line-height: 1.2;
  margin-bottom: 0.25rem;
}

.subtitle {
  font-size: 0.9rem;
  color: var(--text-light);
  font-weight: 500;
  margin: 0;
  letter-spacing: 0.3px;
  text-transform: uppercase;
}

.main-content {
  padding: 1.5rem;
  max-width: 600px;
  margin: 0 auto;
}

.current-weight-card {
  background: linear-gradient(135deg, rgba(129, 140, 248, 0.1) 0%, rgba(167, 139, 250, 0.05) 100%);
  border-radius: 1.25rem;
  padding: 2.5rem;
  margin-bottom: 1.5rem;
  box-shadow: var(--shadow);
  text-align: center;
  border: 1px solid rgba(129, 140, 248, 0.2);
  position: relative;
  overflow: hidden;
}

.current-weight-card::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 4px;
  background: linear-gradient(90deg, var(--primary) 0%, var(--secondary) 50%, var(--accent) 100%);
}

.current-weight-card::after {
  content: '';
  position: absolute;
  top: -50%;
  right: -20%;
  width: 200px;
  height: 200px;
  background: radial-gradient(circle, rgba(129, 140, 248, 0.15) 0%, transparent 70%);
  border-radius: 50%;
  pointer-events: none;
}

.weight-label {
  font-size: 0.9rem;
  color: var(--text-light);
  text-transform: uppercase;
  letter-spacing: 1px;
  margin-bottom: 0.5rem;
  font-weight: 600;
}

.weight-value {
  font-size: 3.5rem;
  font-weight: 700;
  background: linear-gradient(135deg, var(--primary-light) 0%, var(--primary) 50%, var(--secondary) 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  margin-bottom: 0.5rem;
  line-height: 1;
  text-shadow: 0 0 30px rgba(129, 140, 248, 0.3);
}

.weight-unit {
  font-size: 1.5rem;
  font-weight: 400;
  color: var(--text-light);
}

.last-update {
  font-size: 0.85rem;
  color: var(--text-light);
  margin-top: 0.5rem;
}

.form-card,
.chart-card,
.history-card {
  background: linear-gradient(135deg, var(--surface) 0%, rgba(129, 140, 248, 0.03) 100%);
  border-radius: 1.25rem;
  padding: 1.75rem;
  margin-bottom: 1.5rem;
  box-shadow: var(--shadow);
  border: 1px solid rgba(129, 140, 248, 0.15);
  backdrop-filter: blur(10px);
  position: relative;
  overflow: hidden;
}

.form-card::before,
.chart-card::before,
.history-card::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 2px;
  background: linear-gradient(90deg, var(--primary) 0%, var(--secondary) 100%);
  opacity: 0.6;
}

.section-title {
  font-size: 1.25rem;
  font-weight: 700;
  margin-bottom: 1.25rem;
  background: linear-gradient(135deg, var(--primary-light) 0%, var(--primary) 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  letter-spacing: -0.3px;
}

.weight-form {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.input-group {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.input-group label {
  font-size: 0.9rem;
  font-weight: 500;
  color: var(--text);
}

.weight-input {
  width: 100%;
  padding: 1.25rem;
  font-size: 1.25rem;
  border: 2px solid var(--border);
  border-radius: 0.875rem;
  background: var(--surface-elevated);
  color: var(--text);
  transition: all 0.3s ease;
  -webkit-appearance: none;
  appearance: none;
  font-weight: 500;
}

.weight-input:focus {
  outline: none;
  border-color: var(--primary);
  box-shadow: 0 0 0 4px rgba(129, 140, 248, 0.2), 0 0 20px rgba(129, 140, 248, 0.3);
  background: linear-gradient(135deg, var(--surface) 0%, rgba(129, 140, 248, 0.05) 100%);
  transform: translateY(-1px);
}

.weight-input::placeholder {
  color: var(--text-lighter);
}

.btn-primary {
  width: 100%;
  padding: 1.125rem;
  font-size: 1rem;
  font-weight: 600;
  color: white;
  background: linear-gradient(135deg, var(--primary) 0%, var(--secondary) 100%);
  border: none;
  border-radius: 0.875rem;
  cursor: pointer;
  transition: all 0.3s ease;
  box-shadow: var(--shadow), var(--shadow-glow);
  position: relative;
  overflow: hidden;
  z-index: 0;
}

.btn-primary > * {
  position: relative;
  z-index: 1;
}

.btn-primary::before {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  width: 0;
  height: 0;
  border-radius: 50%;
  background: rgba(255, 255, 255, 0.2);
  transform: translate(-50%, -50%);
  transition: width 0.6s, height 0.6s;
}

.btn-primary:hover::before {
  width: 300px;
  height: 300px;
}

.btn-primary:hover {
  transform: translateY(-2px);
  box-shadow: var(--shadow-lg), 0 0 30px rgba(129, 140, 248, 0.5);
}

.btn-primary:active {
  transform: translateY(0);
}

.chart-container {
  position: relative;
  height: 250px;
  width: 100%;
}

.history-list {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.history-item-wrapper {
  position: relative;
  overflow: hidden;
  border-radius: 1rem;
  touch-action: pan-x pan-y;
  user-select: none;
  -webkit-user-select: none;
}

.history-item {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 1.25rem;
  background: var(--surface-elevated);
  border-radius: 1rem;
  transition: transform 0.2s ease-out;
  border: 1px solid var(--border);
  position: relative;
  z-index: 1;
  cursor: grab;
}

.history-item:active {
  cursor: grabbing;
  transition: transform 0.1s ease-out;
}

.history-content {
  display: flex;
  align-items: center;
  gap: 1rem;
  flex: 1;
  min-width: 0;
}

.delete-action {
  position: absolute;
  right: 0;
  top: 0;
  bottom: 0;
  width: 120px;
  background: linear-gradient(90deg, transparent 0%, var(--danger) 100%);
  display: flex;
  align-items: center;
  justify-content: flex-end;
  padding-right: 1.5rem;
  opacity: 0;
  transition: opacity 0.2s ease;
  z-index: 0;
}

.delete-action-visible {
  opacity: 1;
}

.delete-text {
  color: white;
  font-weight: 600;
  font-size: 0.9rem;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.history-date {
  display: flex;
  flex-direction: column;
  align-items: center;
  min-width: 55px;
  padding: 0.75rem 0.5rem;
  background: linear-gradient(135deg, rgba(129, 140, 248, 0.15) 0%, rgba(167, 139, 250, 0.1) 100%);
  border-radius: 0.75rem;
  border: 1px solid rgba(129, 140, 248, 0.3);
  position: relative;
  overflow: hidden;
}

.history-date::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 2px;
  background: linear-gradient(90deg, var(--primary) 0%, var(--secondary) 100%);
}

.date-day {
  font-size: 1.5rem;
  font-weight: 700;
  background: linear-gradient(135deg, var(--primary-light) 0%, var(--primary) 50%, var(--secondary) 100%);
  -webkit-background-clip: text;
  -webkit-text-fill-color: transparent;
  background-clip: text;
  line-height: 1;
  position: relative;
  z-index: 1;
}

.date-month {
  font-size: 0.75rem;
  color: var(--text-light);
  text-transform: capitalize;
  margin-top: 0.25rem;
}

.history-weight {
  flex: 1;
  display: flex;
  align-items: baseline;
  gap: 0.25rem;
}

.weight-number {
  font-size: 1.5rem;
  font-weight: 600;
  color: var(--text);
}

.weight-unit-small {
  font-size: 1rem;
  color: var(--text-light);
}

.history-diff {
  min-width: 60px;
  text-align: right;
}

.diff-badge {
  display: inline-block;
  padding: 0.25rem 0.75rem;
  border-radius: 1rem;
  font-size: 0.85rem;
  font-weight: 600;
}

.diff-positive {
  background: linear-gradient(135deg, rgba(248, 113, 113, 0.2) 0%, rgba(239, 68, 68, 0.15) 100%);
  color: var(--danger);
  border: 1px solid rgba(248, 113, 113, 0.4);
}

.diff-negative {
  background: linear-gradient(135deg, rgba(52, 211, 153, 0.2) 0%, rgba(16, 185, 129, 0.15) 100%);
  color: var(--success);
  border: 1px solid rgba(52, 211, 153, 0.4);
}

.diff-neutral {
  background: linear-gradient(135deg, var(--surface-elevated) 0%, rgba(129, 140, 248, 0.1) 100%);
  color: var(--text-light);
  border: 1px solid rgba(129, 140, 248, 0.2);
}

.empty-state {
  text-align: center;
  padding: 4rem 1.5rem;
  color: var(--text-light);
  background: linear-gradient(135deg, var(--surface) 0%, rgba(129, 140, 248, 0.05) 100%);
  border-radius: 1.25rem;
  border: 1px solid rgba(129, 140, 248, 0.2);
  margin-top: 1.5rem;
  position: relative;
  overflow: hidden;
}

.empty-state::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 2px;
  background: linear-gradient(90deg, var(--primary) 0%, var(--secondary) 100%);
  opacity: 0.6;
}

.empty-icon {
  font-size: 5rem;
  margin-bottom: 1.5rem;
  filter: drop-shadow(0 0 20px rgba(129, 140, 248, 0.3));
  animation: float 3s ease-in-out infinite;
}

@keyframes float {
  0%, 100% {
    transform: translateY(0px);
  }
  50% {
    transform: translateY(-10px);
  }
}

.empty-text {
  font-size: 1.5rem;
  font-weight: 600;
  color: var(--text);
  margin-bottom: 0.75rem;
}

.empty-subtext {
  font-size: 1rem;
  color: var(--text-light);
}

@media (max-width: 480px) {
  .main-content {
    padding: 1rem;
  }

  .title {
    font-size: 1.75rem;
  }

  .weight-value {
    font-size: 2.5rem;
  }
}
</style>

