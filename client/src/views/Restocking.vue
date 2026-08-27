<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budget.title') }}</h3>
        </div>
        <div class="budget-slider-row">
          <input
            type="range"
            min="0"
            :max="maxBudget"
            :step="budgetStep"
            v-model.number="budget"
            class="budget-slider"
          />
          <div class="budget-value">{{ currencySymbol }}{{ budget.toLocaleString() }}</div>
        </div>
        <div class="budget-meta">
          <span>{{ t('restocking.budget.max', { amount: currencySymbol + maxBudget.toLocaleString() }) }}</span>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendations') }} ({{ selectedCount }}/{{ affordableRecommendations.length }})</h3>
        </div>
        <div v-if="affordableRecommendations.length === 0" class="empty-state">
          {{ t('restocking.noRecommendations') }}
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th></th>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.category') }}</th>
                <th>{{ t('restocking.table.onHand') }}</th>
                <th>{{ t('restocking.table.forecastedDemand') }}</th>
                <th>{{ t('restocking.table.recommendedQty') }}</th>
                <th>{{ t('restocking.table.cost') }}</th>
                <th>{{ t('restocking.table.leadTime') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="rec in affordableRecommendations"
                :key="rec.sku"
                :class="{ 'row-disabled': !canAfford(rec) }"
              >
                <td>
                  <input
                    type="checkbox"
                    :checked="selectedSkus.has(rec.sku)"
                    :disabled="!canAfford(rec)"
                    @change="toggleSelected(rec)"
                  />
                </td>
                <td><strong>{{ rec.sku }}</strong></td>
                <td>{{ rec.name }}</td>
                <td>{{ rec.category }}</td>
                <td>{{ rec.quantity_on_hand }}</td>
                <td>{{ rec.forecasted_demand }}</td>
                <td>{{ rec.recommended_quantity }}</td>
                <td>{{ currencySymbol }}{{ rec.total_cost.toLocaleString() }}</td>
                <td>{{ t('restocking.leadTimeDays', { days: rec.lead_time_days }) }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <div class="card order-summary-card">
        <div class="order-summary-row">
          <div>
            <div class="summary-label">{{ t('restocking.selectedTotal') }}</div>
            <div class="summary-value">{{ currencySymbol }}{{ selectedTotalCost.toLocaleString() }} / {{ currencySymbol }}{{ budget.toLocaleString() }}</div>
          </div>
          <button
            class="place-order-btn"
            :disabled="selectedSkus.size === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? t('restocking.placing') : t('restocking.placeOrder') }}
          </button>
        </div>
        <div v-if="orderError" class="error">{{ orderError }}</div>
        <div v-if="orderSuccess" class="success-banner">
          {{ t('restocking.orderPlaced', { orderNumber: orderSuccess.order_number }) }}
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

const { t, currentCurrency } = useI18n()

const currencySymbol = computed(() => {
  return currentCurrency.value === 'JPY' ? '¥' : '$'
})

const loading = ref(true)
const error = ref(null)
const recommendations = ref([])
const selectedSkus = ref(new Set())
const budget = ref(0)
const maxBudget = ref(0)
const submitting = ref(false)
const orderError = ref(null)
const orderSuccess = ref(null)

const budgetStep = computed(() => {
  // Round step to a sensible increment based on the overall scale
  return maxBudget.value > 20000 ? 500 : 100
})

// Sorted by cost descending (highest need first), matches API order
const affordableRecommendations = computed(() => recommendations.value)

const canAfford = (rec) => {
  if (selectedSkus.value.has(rec.sku)) return true
  return selectedTotalCost.value + rec.total_cost <= budget.value
}

const selectedTotalCost = computed(() => {
  return recommendations.value
    .filter(rec => selectedSkus.value.has(rec.sku))
    .reduce((sum, rec) => sum + rec.total_cost, 0)
})

const selectedCount = computed(() => selectedSkus.value.size)

const toggleSelected = (rec) => {
  const next = new Set(selectedSkus.value)
  if (next.has(rec.sku)) {
    next.delete(rec.sku)
  } else {
    next.add(rec.sku)
  }
  selectedSkus.value = next
}

const loadRecommendations = async () => {
  try {
    loading.value = true
    error.value = null
    const data = await api.getRestockingRecommendations()
    recommendations.value = data

    // Budget slider max = total cost of restocking everything recommended
    const totalNeed = data.reduce((sum, rec) => sum + rec.total_cost, 0)
    maxBudget.value = Math.ceil(totalNeed / 100) * 100
    budget.value = Math.round(maxBudget.value / 2)
  } catch (err) {
    error.value = 'Failed to load restocking recommendations: ' + err.message
  } finally {
    loading.value = false
  }
}

const placeOrder = async () => {
  if (selectedSkus.value.size === 0) return

  submitting.value = true
  orderError.value = null
  orderSuccess.value = null

  try {
    const items = recommendations.value
      .filter(rec => selectedSkus.value.has(rec.sku))
      .map(rec => ({
        sku: rec.sku,
        name: rec.name,
        category: rec.category,
        quantity: rec.recommended_quantity,
        unit_cost: rec.unit_cost,
        total_cost: rec.total_cost
      }))

    const order = await api.createRestockingOrder({ items, budget: budget.value })
    orderSuccess.value = order
    selectedSkus.value = new Set()
  } catch (err) {
    orderError.value = 'Failed to place order: ' + (err.response?.data?.detail || err.message)
  } finally {
    submitting.value = false
  }
}

onMounted(loadRecommendations)
</script>

<style scoped>
.budget-card {
  margin-bottom: 1.5rem;
}

.budget-slider-row {
  display: flex;
  align-items: center;
  gap: 1.5rem;
  padding: 0.5rem 0;
}

.budget-slider {
  flex: 1;
  height: 6px;
  accent-color: #2563eb;
}

.budget-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  min-width: 140px;
  text-align: right;
}

.budget-meta {
  font-size: 0.813rem;
  color: #64748b;
  margin-top: 0.5rem;
}

.row-disabled {
  opacity: 0.45;
}

.empty-state {
  padding: 2rem;
  text-align: center;
  color: #64748b;
}

.order-summary-card {
  margin-top: 1.5rem;
}

.order-summary-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 0.5rem 0;
}

.summary-label {
  font-size: 0.813rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.summary-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  margin-top: 0.25rem;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  padding: 0.75rem 1.75rem;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
}

.success-banner {
  margin-top: 1rem;
  padding: 0.75rem 1rem;
  background: #d1fae5;
  color: #065f46;
  border-radius: 8px;
  font-size: 0.875rem;
  font-weight: 500;
}
</style>
