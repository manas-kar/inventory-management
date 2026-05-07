<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Recommend items to restock within your available budget</p>
    </div>

    <!-- Budget card -->
    <div class="card budget-card">
      <div class="budget-label">Available Budget</div>
      <input
        type="range"
        class="budget-slider"
        min="0"
        max="50000"
        step="500"
        :value="budget"
        @input="onBudgetInput"
      />
      <div class="budget-amount">${{ budget.toLocaleString() }}</div>
      <div class="budget-sublabel">Drag to adjust budget</div>
    </div>

    <!-- Success banner -->
    <div v-if="successOrder" class="success-banner">
      <span>
        Order <strong>{{ successOrder.order_number }}</strong> placed successfully
        &middot;
        Expected delivery: <strong>{{ formatDate(successOrder.expected_delivery) }}</strong>
      </span>
      <button class="dismiss-btn" @click="successOrder = null">Dismiss</button>
    </div>

    <!-- Summary bar -->
    <div v-if="summary" class="summary-bar">
      <span>
        <strong>{{ summary.items_included }}</strong> item{{ summary.items_included !== 1 ? 's' : '' }} recommended
        &nbsp;&middot;&nbsp;
        Total cost: <strong>${{ summary.total_cost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</strong>
        &nbsp;&middot;&nbsp;
        Budget remaining: <strong>${{ (summary.budget - summary.total_cost).toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</strong>
      </span>
    </div>

    <!-- Recommendations card -->
    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Recommended Items ({{ summary ? summary.items_included : 0 }} within budget)</h3>
      </div>

      <div v-if="loading" class="loading">Loading recommendations...</div>
      <div v-else-if="error" class="error">{{ error }}</div>
      <div v-else-if="recommendations.length === 0" class="empty-state">
        All items are sufficiently stocked.
      </div>
      <div v-else class="table-container">
        <table>
          <thead>
            <tr>
              <th>Priority</th>
              <th>Item</th>
              <th>Warehouse</th>
              <th>Trend</th>
              <th>Stock Level</th>
              <th>Restock Qty</th>
              <th>Cost</th>
              <th>Status</th>
            </tr>
          </thead>
          <tbody>
            <tr
              v-for="item in recommendations"
              :key="item.sku"
              :class="{ 'row-excluded': !item.included }"
            >
              <td>
                <span :class="['badge', getPriorityClass(item.priority)]">
                  {{ item.priority }}
                </span>
              </td>
              <td>
                <div class="item-name">{{ item.name }}</div>
                <div class="item-sku">{{ item.sku }}</div>
              </td>
              <td>{{ item.warehouse }}</td>
              <td>
                <span :class="['badge', item.trend]">{{ item.trend }}</span>
              </td>
              <td>
                <div class="stock-info">
                  <span class="stock-text">{{ item.quantity_on_hand }} / {{ item.reorder_point }}</span>
                  <div class="stock-bar-track">
                    <div
                      class="stock-bar-fill"
                      :style="{ width: getStockPercent(item) + '%' }"
                      :class="getStockBarClass(item)"
                    ></div>
                  </div>
                </div>
              </td>
              <td>{{ item.restock_quantity }}</td>
              <td>${{ item.restock_cost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</td>
              <td>
                <span v-if="item.included" class="status-included">Within budget</span>
                <span v-else class="status-excluded">Over budget</span>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <!-- Place order section -->
    <div class="place-order-section">
      <div v-if="placeError" class="error place-error">{{ placeError }}</div>
      <button
        class="place-order-btn"
        :disabled="!summary || !summary.items_included || placing"
        @click="placeOrder"
      >
        {{ placing ? 'Placing Order...' : 'Place Restocking Order' }}
      </button>
    </div>
  </div>
</template>

<script>
import { ref, onMounted } from 'vue'
import axios from 'axios'

const API_BASE = 'http://localhost:8001/api'

export default {
  name: 'Restocking',
  setup() {
    const budget = ref(25000)
    const recommendations = ref([])
    const summary = ref(null)
    const loading = ref(false)
    const error = ref(null)
    const placing = ref(false)
    const placeError = ref(null)
    const successOrder = ref(null)

    // Debounce timer reference for slider input
    let debounceTimer = null

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        const response = await axios.get(`${API_BASE}/restocking/recommendations?budget=${budget.value}`)
        const data = response.data
        recommendations.value = data.recommendations
        summary.value = {
          budget: data.budget,
          total_cost: data.total_cost,
          items_included: data.items_included,
          items_excluded: data.items_excluded
        }
      } catch (err) {
        error.value = 'Failed to load recommendations'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    // Debounce slider changes by 400ms before fetching to avoid spamming the API
    const onBudgetInput = (event) => {
      budget.value = Number(event.target.value)
      clearTimeout(debounceTimer)
      debounceTimer = setTimeout(() => {
        loadRecommendations()
      }, 400)
    }

    const getPriorityClass = (priority) => {
      const map = { high: 'danger', medium: 'warning', low: 'info' }
      return map[priority] || 'info'
    }

    const getStockPercent = (item) => {
      if (!item.reorder_point) return 0
      return Math.min(100, Math.round((item.quantity_on_hand / item.reorder_point) * 100))
    }

    const getStockBarClass = (item) => {
      const pct = getStockPercent(item)
      if (pct < 50) return 'bar-red'
      if (pct < 75) return 'bar-yellow'
      return 'bar-green'
    }

    const formatDate = (str) => {
      return new Date(str).toLocaleDateString('en-US', {
        year: 'numeric',
        month: 'short',
        day: 'numeric'
      })
    }

    const placeOrder = async () => {
      placeError.value = null
      placing.value = true
      try {
        const items = recommendations.value
          .filter(r => r.included)
          .map(r => ({
            sku: r.sku,
            name: r.name,
            category: r.category,
            warehouse: r.warehouse,
            quantity: r.restock_quantity,
            unit_cost: r.unit_cost
          }))

        const response = await axios.post(`${API_BASE}/restocking/orders`, {
          items,
          budget: budget.value
        })
        successOrder.value = response.data
      } catch (err) {
        placeError.value = 'Failed to place order. Please try again.'
        console.error(err)
      } finally {
        placing.value = false
      }
    }

    onMounted(() => loadRecommendations())

    return {
      budget,
      recommendations,
      summary,
      loading,
      error,
      placing,
      placeError,
      successOrder,
      onBudgetInput,
      getPriorityClass,
      getStockPercent,
      getStockBarClass,
      formatDate,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding-bottom: 2rem;
}

/* Budget card */
.budget-card {
  max-width: 540px;
  margin-bottom: 1.25rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 1rem;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  cursor: pointer;
  margin-bottom: 0.75rem;
}

.budget-amount {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  margin-bottom: 0.25rem;
}

.budget-sublabel {
  font-size: 0.813rem;
  color: #94a3b8;
}

/* Success banner */
.success-banner {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 10px;
  padding: 0.875rem 1.25rem;
  margin-bottom: 1.25rem;
  color: #065f46;
  font-size: 0.938rem;
}

.dismiss-btn {
  background: none;
  border: 1px solid #6ee7b7;
  color: #065f46;
  border-radius: 6px;
  padding: 0.25rem 0.75rem;
  font-size: 0.813rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s;
}

.dismiss-btn:hover {
  background: #a7f3d0;
}

/* Summary bar */
.summary-bar {
  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  padding: 0.75rem 1.25rem;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
  color: #334155;
}

/* Empty state */
.empty-state {
  padding: 3rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

/* Table rows — excluded items are dimmed */
.row-excluded {
  opacity: 0.4;
}

/* Item cell */
.item-name {
  font-weight: 600;
  color: #0f172a;
  font-size: 0.875rem;
}

.item-sku {
  font-size: 0.75rem;
  color: #64748b;
  margin-top: 0.125rem;
}

/* Stock level cell */
.stock-info {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  min-width: 100px;
}

.stock-text {
  font-size: 0.813rem;
  color: #334155;
}

.stock-bar-track {
  height: 5px;
  background: #e2e8f0;
  border-radius: 99px;
  overflow: hidden;
  width: 100%;
}

.stock-bar-fill {
  height: 100%;
  border-radius: 99px;
  transition: width 0.3s ease;
}

.bar-red {
  background: #dc2626;
}

.bar-yellow {
  background: #d97706;
}

.bar-green {
  background: #059669;
}

/* Status text */
.status-included {
  color: #059669;
  font-weight: 600;
  font-size: 0.813rem;
}

.status-excluded {
  color: #94a3b8;
  font-size: 0.813rem;
}

/* Place order section */
.place-order-section {
  margin-top: 0.5rem;
}

.place-error {
  margin-bottom: 0.75rem;
}

.place-order-btn {
  width: 100%;
  padding: 0.875rem 1.5rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 10px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
  letter-spacing: -0.01em;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}
</style>
