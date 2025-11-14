<template>
  <canvas ref="chartCanvas"></canvas>
</template>

<script>
import { ref, onMounted, watch } from 'vue'
import {
  Chart as ChartJS,
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend,
  Filler,
  LineController
} from 'chart.js'
ChartJS.register(
  CategoryScale,
  LinearScale,
  PointElement,
  LineElement,
  Title,
  Tooltip,
  Legend,
  Filler,
  LineController
)

export default {
  name: 'WeightChart',
  props: {
    data: {
      type: Object,
      required: true
    }
  },
  setup(props) {
    const chartCanvas = ref(null)
    let chartInstance = null

    const createChart = () => {
      if (!chartCanvas.value) return

      const ctx = chartCanvas.value.getContext('2d')

      if (chartInstance) {
        chartInstance.destroy()
      }

      const weights = props.data.weights || []
      const labels = props.data.labels || []

      if (weights.length === 0) return

      // Calcular el rango del eje Y con un margen
      const minWeight = Math.min(...weights)
      const maxWeight = Math.max(...weights)
      const range = maxWeight - minWeight
      const margin = range * 0.2 || 2

      // Crear gradiente para la línea
      const gradient = ctx.createLinearGradient(0, 0, 0, 400)
      gradient.addColorStop(0, 'rgba(129, 140, 248, 0.4)')
      gradient.addColorStop(0.5, 'rgba(129, 140, 248, 0.2)')
      gradient.addColorStop(1, 'rgba(129, 140, 248, 0.05)')

      chartInstance = new ChartJS(ctx, {
        type: 'line',
        data: {
          labels: labels,
          datasets: [
            {
              label: 'Peso (kg)',
              data: weights,
              borderColor: '#818cf8',
              backgroundColor: gradient,
              borderWidth: 3,
              fill: true,
              tension: 0.5,
              pointRadius: 6,
              pointHoverRadius: 8,
              pointBackgroundColor: '#818cf8',
              pointBorderColor: '#1e293b',
              pointBorderWidth: 3,
              pointHoverBackgroundColor: '#a5b4fc',
              pointHoverBorderColor: '#1e293b',
              pointHoverBorderWidth: 4,
              shadowOffsetX: 0,
              shadowOffsetY: 4,
              shadowBlur: 10,
              shadowColor: 'rgba(129, 140, 248, 0.3)'
            }
          ]
        },
        options: {
          responsive: true,
          maintainAspectRatio: false,
          plugins: {
            legend: {
              display: false
            },
            tooltip: {
              backgroundColor: 'rgba(30, 41, 59, 0.95)',
              padding: 14,
              titleFont: {
                size: 14,
                weight: 'bold',
                family: '-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif'
              },
              bodyFont: {
                size: 14,
                family: '-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif'
              },
              displayColors: false,
              borderColor: '#818cf8',
              borderWidth: 1,
              cornerRadius: 8,
              callbacks: {
                label: function(context) {
                  return `Peso: ${context.parsed.y.toFixed(1)} kg`
                },
                title: function(context) {
                  return context[0].label
                }
              }
            }
          },
          scales: {
            x: {
              grid: {
                display: false,
                drawBorder: false
              },
              ticks: {
                font: {
                  size: 11,
                  family: '-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif'
                },
                color: '#94a3b8'
              }
            },
            y: {
              min: minWeight - margin,
              max: maxWeight + margin,
              grid: {
                color: 'rgba(51, 65, 85, 0.3)',
                drawBorder: false,
                lineWidth: 1
              },
              ticks: {
                font: {
                  size: 11,
                  family: '-apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif'
                },
                color: '#94a3b8',
                callback: function(value) {
                  return value.toFixed(1) + ' kg'
                }
              }
            }
          },
          interaction: {
            intersect: false,
            mode: 'index'
          },
          elements: {
            point: {
              hoverBorderWidth: 4
            }
          }
        }
      })
    }

    onMounted(() => {
      createChart()
    })

    watch(
      () => props.data,
      () => {
        createChart()
      },
      { deep: true }
    )

    return {
      chartCanvas
    }
  }
}
</script>

<style scoped>
canvas {
  max-width: 100%;
  height: 100% !important;
}
</style>

