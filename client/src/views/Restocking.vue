<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div class="card budget-card">
      <div class="budget-row">
        <label class="budget-label" for="budget-slider">
          {{ t('restocking.budgetLabel') }}: <strong>{{ currencySymbol }}{{ budget.toLocaleString() }}</strong>
        </label>
        <input
          id="budget-slider"
          type="range"
          min="0"
          :max="maxBudget"
          step="500"
          v-model.number="budget"
          class="budget-slider"
        />
      </div>

      <div class="stats-grid">
        <div class="stat-card info">
          <div class="stat-label">{{ t('restocking.itemsRecommended', { count: recommendations.length }) }}</div>
          <div class="stat-value">{{ recommendations.length }}</div>
        </div>
        <div class="stat-card warning">
          <div class="stat-label">{{ t('restocking.totalCost') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ totalCost.toLocaleString() }}</div>
        </div>
        <div class="stat-card success">
          <div class="stat-label">{{ t('restocking.remainingBudget') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ remainingBudget.toLocaleString() }}</div>
        </div>
      </div>
    </div>

    <div v-if="orderConfirmation" class="success-banner">
      {{ t('restocking.orderPlaced', { orderNumber: orderConfirmation.order_number }) }}
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else class="card">
      <div class="card-header">
        <h3 class="card-title">{{ t('restocking.recommended') }}</h3>
      </div>

      <div v-if="recommendations.length === 0" class="empty-state">
        {{ t('restocking.noItems') }}
      </div>

      <div v-else class="table-container">
        <table>
          <thead>
            <tr>
              <th>{{ t('restocking.table.item') }}</th>
              <th>{{ t('restocking.table.sku') }}</th>
              <th>{{ t('restocking.table.category') }}</th>
              <th>{{ t('restocking.table.trend') }}</th>
              <th>{{ t('restocking.table.currentDemand') }}</th>
              <th>{{ t('restocking.table.forecastedDemand') }}</th>
              <th>{{ t('restocking.table.quantity') }}</th>
              <th>{{ t('restocking.table.unitCost') }}</th>
              <th>{{ t('restocking.table.lineTotal') }}</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in recommendations" :key="item.sku">
              <td>{{ translateProductName(item.name) }}</td>
              <td>{{ item.sku }}</td>
              <td>{{ item.category }}</td>
              <td><span :class="['badge', item.trend]">{{ t(`trends.${item.trend}`) }}</span></td>
              <td>{{ item.current_demand.toLocaleString() }}</td>
              <td>{{ item.forecasted_demand.toLocaleString() }}</td>
              <td><strong>{{ item.recommended_quantity.toLocaleString() }}</strong></td>
              <td>{{ currencySymbol }}{{ item.unit_cost.toFixed(2) }}</td>
              <td>{{ currencySymbol }}{{ item.line_total.toLocaleString() }}</td>
            </tr>
          </tbody>
        </table>
      </div>

      <div class="place-order-row">
        <button
          class="place-order-btn"
          :disabled="recommendations.length === 0 || placing"
          @click="placeOrder"
        >
          {{ placing ? t('restocking.placing') : t('restocking.placeOrder') }}
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted, watch } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency, translateProductName } = useI18n()

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const maxBudget = 100000
    const budget = ref(25000)
    const recommendations = ref([])
    const totalCost = ref(0)
    const remainingBudget = ref(budget.value)
    const loading = ref(true)
    const error = ref(null)
    const placing = ref(false)
    const orderConfirmation = ref(null)

    let debounceTimer = null

    const loadRecommendations = async () => {
      try {
        loading.value = true
        error.value = null
        const data = await api.getRestockRecommendations(budget.value)
        recommendations.value = data.items
        totalCost.value = data.total_cost
        remainingBudget.value = data.remaining_budget
      } catch (err) {
        error.value = 'Failed to load recommendations: ' + err.message
      } finally {
        loading.value = false
      }
    }

    // Debounce so dragging the slider doesn't fire a request per pixel
    watch(budget, () => {
      orderConfirmation.value = null
      clearTimeout(debounceTimer)
      debounceTimer = setTimeout(loadRecommendations, 300)
    })

    const placeOrder = async () => {
      try {
        placing.value = true
        const payload = {
          budget: budget.value,
          items: recommendations.value.map(item => ({
            sku: item.sku,
            name: item.name,
            quantity: item.recommended_quantity,
            unit_cost: item.unit_cost
          }))
        }
        orderConfirmation.value = await api.submitRestockOrder(payload)
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        placing.value = false
      }
    }

    onMounted(loadRecommendations)

    return {
      t,
      budget,
      maxBudget,
      recommendations,
      totalCost,
      remainingBudget,
      loading,
      error,
      placing,
      orderConfirmation,
      placeOrder,
      currencySymbol,
      translateProductName
    }
  }
}
</script>

<style scoped>
.budget-card {
  margin-bottom: 1.5rem;
}

.budget-row {
  margin-bottom: 1.25rem;
}

.budget-label {
  display: block;
  font-size: 0.938rem;
  font-weight: 600;
  color: #0f172a;
  margin-bottom: 0.75rem;
}

.budget-slider {
  width: 100%;
  height: 6px;
  border-radius: 999px;
  background: #e2e8f0;
  appearance: none;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  appearance: none;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 3px solid white;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.3);
}

.budget-slider::-moz-range-thumb {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 3px solid white;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.3);
}

.empty-state {
  text-align: center;
  padding: 2rem;
  color: #64748b;
  font-size: 0.938rem;
}

.place-order-row {
  display: flex;
  justify-content: flex-end;
  margin-top: 1.25rem;
  padding-top: 1.25rem;
  border-top: 1px solid #e2e8f0;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.75rem 1.75rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #a7f3d0;
  color: #065f46;
  padding: 1rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
  font-weight: 500;
}
</style>
