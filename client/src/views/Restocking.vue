<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Manage inventory restocking recommendations and place orders</p>
    </div>

    <div class="card budget-card">
      <div class="card-header">
        <h3 class="card-title">Budget</h3>
      </div>
      <div class="budget-controls">
        <div class="budget-slider-row">
          <label class="budget-label" for="budget-slider">Restocking Budget</label>
          <span class="budget-value">{{ formatCurrency(budget) }}</span>
        </div>
        <input
          id="budget-slider"
          type="range"
          class="budget-slider"
          min="0"
          max="50000"
          step="500"
          v-model.number="budget"
        />
        <div class="budget-summary">
          <span class="budget-spend">Recommended spend: {{ formatCurrency(recommendedCost) }}</span>
          <span :class="['budget-remaining', { 'over': remainingBudget < 0 }]">
            Remaining: {{ formatCurrency(remainingBudget) }}
          </span>
        </div>
      </div>
    </div>

    <div class="card">
      <div class="card-header">
        <h3 class="card-title">Recommendations</h3>
      </div>

      <div v-if="loading" class="loading">Loading recommendations...</div>
      <div v-else-if="error" class="error">{{ error }}</div>
      <div v-else>
        <div class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th class="col-item">Item</th>
                <th class="col-sku">SKU</th>
                <th class="col-trend">Trend</th>
                <th class="col-qty">Restock Qty</th>
                <th class="col-unit-cost">Unit Cost</th>
                <th class="col-total-cost">Total Cost</th>
                <th class="col-include">Include</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.id"
                :class="{ 'row-over-budget': isRowOverBudget(item) }"
              >
                <td class="col-item"><strong>{{ item.item_name }}</strong></td>
                <td class="col-sku"><span class="sku-tag">{{ item.item_sku }}</span></td>
                <td class="col-trend">
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td class="col-qty">{{ item.restock_quantity.toLocaleString() }}</td>
                <td class="col-unit-cost">{{ formatCurrency(item.unit_cost) }}</td>
                <td class="col-total-cost"><strong>{{ formatCurrency(item.total_cost) }}</strong></td>
                <td class="col-include">
                  <input
                    type="checkbox"
                    :checked="checkedIds.has(item.id)"
                    @change="toggleItem(item.id)"
                    class="include-checkbox"
                  />
                </td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="selection-summary">
          <span>{{ selectedItems.length }} item{{ selectedItems.length !== 1 ? 's' : '' }} selected &mdash; Total: <strong>{{ formatCurrency(totalCost) }}</strong></span>
          <span v-if="overBudget" class="over-budget-warning">Exceeds budget by {{ formatCurrency(totalCost - budget) }}</span>
        </div>
      </div>
    </div>

    <div class="order-actions">
      <div v-if="successMessage" class="success-message">{{ successMessage }}</div>
      <button
        class="btn-place-order"
        :disabled="selectedItems.length === 0 || overBudget || placingOrder"
        @click="placeOrder"
      >
        {{ placingOrder ? 'Placing Order...' : 'Place Restocking Order' }}
      </button>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const budget = ref(25000)
    const recommendations = ref([])
    const checkedIds = ref(new Set())
    const loading = ref(false)
    const error = ref(null)
    const successMessage = ref(null)
    const placingOrder = ref(false)

    const formatCurrency = (value) => {
      return value.toLocaleString('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 0 })
    }

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getRestockingRecommendations(budget.value)
        recommendations.value = data
        // Pre-check items where recommended === true
        const newChecked = new Set()
        data.forEach(item => {
          if (item.recommended) newChecked.add(item.id)
        })
        checkedIds.value = newChecked
      } catch (err) {
        error.value = 'Failed to load recommendations'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const toggleItem = (id) => {
      const next = new Set(checkedIds.value)
      if (next.has(id)) {
        next.delete(id)
      } else {
        next.add(id)
      }
      checkedIds.value = next
    }

    const selectedItems = computed(() => {
      return recommendations.value.filter(item => checkedIds.value.has(item.id))
    })

    const totalCost = computed(() => {
      return selectedItems.value.reduce((sum, item) => sum + item.total_cost, 0)
    })

    const recommendedCost = computed(() => {
      return recommendations.value
        .filter(item => item.recommended)
        .reduce((sum, item) => sum + item.total_cost, 0)
    })

    const remainingBudget = computed(() => {
      return budget.value - recommendedCost.value
    })

    const overBudget = computed(() => {
      return totalCost.value > budget.value
    })

    // Determine if a row is over budget by checking cumulative cost of checked items
    const isRowOverBudget = (item) => {
      if (!checkedIds.value.has(item.id)) return false
      let cumulative = 0
      for (const rec of recommendations.value) {
        if (!checkedIds.value.has(rec.id)) continue
        cumulative += rec.total_cost
        if (rec.id === item.id) break
      }
      return cumulative > budget.value
    }

    const placeOrder = async () => {
      if (selectedItems.value.length === 0 || overBudget.value) return
      placingOrder.value = true
      successMessage.value = null
      try {
        const items = selectedItems.value.map(item => ({
          sku: item.item_sku,
          name: item.item_name,
          quantity: item.restock_quantity,
          unit_price: item.unit_cost
        }))
        const result = await api.placeRestockingOrder({
          items,
          total_value: totalCost.value
        })
        successMessage.value = `Order ${result.order_number} placed. Expected delivery in 7 days.`
        // Uncheck all and refresh
        checkedIds.value = new Set()
        await loadRecommendations()
      } catch (err) {
        error.value = 'Failed to place order'
        console.error(err)
      } finally {
        placingOrder.value = false
      }
    }

    watch(budget, () => {
      checkedIds.value = new Set()
      loadRecommendations()
    })

    onMounted(() => loadRecommendations())

    return {
      budget,
      recommendations,
      checkedIds,
      loading,
      error,
      successMessage,
      placingOrder,
      selectedItems,
      totalCost,
      recommendedCost,
      remainingBudget,
      overBudget,
      formatCurrency,
      toggleItem,
      isRowOverBudget,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 0;
}

.budget-card {
  margin-bottom: 1.25rem;
}

.budget-controls {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.budget-slider-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.budget-label {
  font-size: 0.938rem;
  font-weight: 600;
  color: #0f172a;
}

.budget-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #2563eb;
  letter-spacing: -0.025em;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  height: 6px;
  cursor: pointer;
}

.budget-summary {
  display: flex;
  justify-content: space-between;
  font-size: 0.875rem;
  color: #64748b;
}

.budget-spend {
  font-weight: 500;
}

.budget-remaining {
  font-weight: 600;
  color: #059669;
}

.budget-remaining.over {
  color: #dc2626;
}

.restock-table {
  table-layout: fixed;
  width: 100%;
}

.col-item { width: 220px; }
.col-sku { width: 130px; }
.col-trend { width: 110px; }
.col-qty { width: 110px; }
.col-unit-cost { width: 110px; }
.col-total-cost { width: 120px; }
.col-include { width: 80px; text-align: center; }

.sku-tag {
  font-size: 0.75rem;
  font-family: monospace;
  background: #f1f5f9;
  color: #475569;
  padding: 0.125rem 0.5rem;
  border-radius: 4px;
}

.include-checkbox {
  width: 16px;
  height: 16px;
  cursor: pointer;
  accent-color: #2563eb;
}

.row-over-budget {
  background: #fff5f5 !important;
}

.row-over-budget td {
  color: #991b1b;
}

.selection-summary {
  display: flex;
  align-items: center;
  gap: 1.5rem;
  padding: 0.875rem 0.75rem;
  border-top: 1px solid #e2e8f0;
  font-size: 0.875rem;
  color: #475569;
  margin-top: 0.5rem;
}

.over-budget-warning {
  color: #dc2626;
  font-weight: 600;
}

.order-actions {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 0.75rem;
  margin-bottom: 2rem;
}

.success-message {
  background: #d1fae5;
  color: #065f46;
  border: 1px solid #a7f3d0;
  border-radius: 8px;
  padding: 0.75rem 1.25rem;
  font-size: 0.938rem;
  font-weight: 500;
  width: 100%;
  text-align: center;
}

.btn-place-order {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  padding: 0.75rem 2rem;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease, opacity 0.2s ease;
}

.btn-place-order:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-place-order:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}
</style>
