<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Recommend items to restock based on demand forecast trends and your available budget</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <div v-if="orderPlaced" class="success-banner">
        Order submitted successfully —
        <router-link to="/orders" class="success-link">view it in the Orders tab</router-link>
      </div>

      <div v-if="orderError" class="error">{{ orderError }}</div>

      <!-- Budget Section -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Available Budget</h3>
        </div>
        <div class="budget-body">
          <div class="slider-row">
            <span class="slider-label">{{ currencySymbol }}0</span>
            <input
              type="range"
              :min="0"
              :max="totalNeeded"
              :step="100"
              v-model.number="budget"
              class="budget-slider"
            />
            <span class="slider-label">{{ currencySymbol }}{{ totalNeeded.toLocaleString() }}</span>
          </div>
          <div class="budget-summary">
            Budget: <strong>{{ currencySymbol }}{{ budget.toLocaleString() }}</strong>
            &mdash;
            Covers <strong>{{ coveredItems.length }}</strong> of <strong>{{ recommendedItems.length }}</strong> recommended items
          </div>
          <div class="budget-actions">
            <button
              class="btn-primary"
              :class="{ disabled: coveredItems.length === 0 || submitting || orderPlaced }"
              :disabled="coveredItems.length === 0 || submitting || orderPlaced"
              @click="placeOrder"
            >
              {{ submitting ? 'Submitting...' : 'Place Order' }}
            </button>
          </div>
        </div>
      </div>

      <!-- Recommended Items Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items ({{ recommendedItems.length }} items)</h3>
        </div>
        <div v-if="recommendedItems.length === 0" class="empty-state">
          No items require restocking at this time.
        </div>
        <div v-else class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th>Item Name</th>
                <th>SKU</th>
                <th>Trend</th>
                <th class="col-number">Qty to Restock</th>
                <th class="col-number">Unit Cost</th>
                <th class="col-number">Restock Cost</th>
                <th>Status</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.item_sku">
                <td>{{ item.item_name }}</td>
                <td class="sku-cell">{{ item.item_sku }}</td>
                <td>
                  <span :class="['badge', item.trend]">
                    {{ item.trend === 'increasing' ? 'Increasing' : 'Stable' }}
                  </span>
                </td>
                <td class="col-number">{{ item.restock_qty.toLocaleString() }}</td>
                <td class="col-number">{{ currencySymbol }}{{ item.inventory.unit_cost.toLocaleString() }}</td>
                <td class="col-number">
                  <strong>{{ currencySymbol }}{{ item.restock_cost.toLocaleString() }}</strong>
                </td>
                <td>
                  <span :class="['badge', isCovered(item) ? 'success' : 'neutral']">
                    {{ isCovered(item) ? 'Covered' : 'Not Covered' }}
                  </span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

const TREND_PRIORITY = { increasing: 0, stable: 1 }

export default {
  name: 'Restocking',
  setup() {
    const { currentCurrency } = useI18n()

    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    const loading = ref(true)
    const error = ref(null)
    const demandForecasts = ref([])
    const inventoryItems = ref([])

    const submitting = ref(false)
    const orderPlaced = ref(false)
    const orderError = ref(null)

    const budget = ref(0)

    const inventoryMap = computed(() => new Map(inventoryItems.value.map(i => [i.sku, i])))

    const recommendedItems = computed(() => {
      return demandForecasts.value
        .filter(f => f.trend !== 'decreasing')
        .map(f => {
          const inv = inventoryMap.value.get(f.item_sku)
          if (!inv) return null
          const deficit = inv.reorder_point - inv.quantity_on_hand
          const restock_qty = deficit > 0
            ? deficit
            : (f.trend === 'increasing' ? Math.round(inv.reorder_point * 0.25) : 0)
          if (restock_qty === 0) return null
          return {
            ...f,
            inventory: inv,
            restock_qty,
            restock_cost: restock_qty * inv.unit_cost
          }
        })
        .filter(Boolean)
        .sort((a, b) => TREND_PRIORITY[a.trend] - TREND_PRIORITY[b.trend])
    })

    const totalNeeded = computed(() =>
      recommendedItems.value.reduce((sum, item) => sum + item.restock_cost, 0)
    )

    const coveredItems = computed(() => {
      let remaining = budget.value
      return recommendedItems.value.filter(item => {
        if (item.restock_cost <= remaining) {
          remaining -= item.restock_cost
          return true
        }
        return false
      })
    })

    const coveredSkuSet = computed(() => new Set(coveredItems.value.map(i => i.item_sku)))

    const isCovered = (item) => coveredSkuSet.value.has(item.item_sku)

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({})
        ])
        demandForecasts.value = forecasts
        inventoryItems.value = inventory
        // Set initial budget to cover all recommended items
        budget.value = totalNeeded.value
      } catch (err) {
        error.value = 'Failed to load restocking data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (coveredItems.value.length === 0 || submitting.value || orderPlaced.value) return
      submitting.value = true
      orderError.value = null
      try {
        const payload = {
          customer: 'Internal Restocking',
          items: coveredItems.value.map(item => ({
            sku: item.item_sku,
            name: item.item_name,
            quantity: item.restock_qty,
            unit_price: item.inventory.unit_cost
          })),
          warehouse: null,
          category: null
        }
        await api.createOrder(payload)
        orderPlaced.value = true
      } catch (err) {
        orderError.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadData)

    return {
      loading,
      error,
      currencySymbol,
      budget,
      totalNeeded,
      recommendedItems,
      coveredItems,
      isCovered,
      submitting,
      orderPlaced,
      orderError,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  /* inherits .main-content padding from App.vue */
}

.success-banner {
  background: #dcfce7;
  border: 1px solid #16a34a;
  border-radius: 8px;
  padding: 1rem 1.25rem;
  margin-bottom: 1.25rem;
  color: #15803d;
  font-size: 0.938rem;
  font-weight: 500;
}

.success-link {
  color: #15803d;
  font-weight: 700;
  text-decoration: underline;
}

.success-link:hover {
  color: #166534;
}

/* Budget card body */
.budget-body {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.slider-row {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.slider-label {
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 500;
  white-space: nowrap;
  min-width: 4rem;
}

.slider-label:last-child {
  text-align: right;
}

.budget-slider {
  flex: 1;
  height: 6px;
  border-radius: 3px;
  accent-color: #2563eb;
  cursor: pointer;
}

.budget-summary {
  font-size: 0.938rem;
  color: #334155;
}

.budget-actions {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.btn-primary {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  padding: 0.625rem 1.5rem;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s ease;
}

.btn-primary:hover:not(.disabled) {
  background: #1d4ed8;
}

.btn-primary.disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Table */
.restocking-table {
  width: 100%;
  border-collapse: collapse;
}

.col-number {
  text-align: right;
}

.sku-cell {
  font-family: 'SF Mono', 'Roboto Mono', 'Fira Code', monospace;
  font-size: 0.813rem;
  color: #64748b;
}

/* Neutral badge for "Not Covered" */
.badge.neutral {
  background: #f1f5f9;
  color: #64748b;
}

.empty-state {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}
</style>
