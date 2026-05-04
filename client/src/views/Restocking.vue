<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Place restocking orders based on demand forecasts and available budget</p>
    </div>

    <div class="card budget-card">
      <div class="card-header">
        <h3 class="card-title">Select Budget</h3>
        <div class="budget-display">${{ selectedBudget.toLocaleString() }}</div>
      </div>
      <div class="budget-slider-container">
        <input
          type="range"
          v-model.number="selectedBudget"
          :min="budgetOptions[0]"
          :max="budgetOptions[budgetOptions.length - 1]"
          :step="budgetStep"
          class="budget-slider"
          @input="handleBudgetChange"
        />
        <div class="budget-labels">
          <span v-for="option in budgetOptions" :key="option" class="budget-label">
            ${{ (option / 1000).toFixed(0) }}K
          </span>
        </div>
      </div>
    </div>

    <div v-if="loading" class="loading">Loading recommendations...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else-if="recommendations && recommendations.length > 0">
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items ({{ recommendations.length }})</h3>
          <div class="budget-summary">
            <span class="budget-used">Used: ${{ totalCost.toLocaleString() }}</span>
            <span class="budget-remaining">Remaining: ${{ budgetRemaining.toLocaleString() }}</span>
          </div>
        </div>
        <div class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Warehouse</th>
                <th>Current Stock</th>
                <th>Forecasted Demand</th>
                <th>Quantity to Order</th>
                <th>Unit Cost</th>
                <th>Total Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.item_id">
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.name }}</td>
                <td>{{ item.warehouse }}</td>
                <td>{{ item.current_stock }}</td>
                <td>{{ item.forecasted_demand }}</td>
                <td><strong>{{ item.quantity_to_order }}</strong></td>
                <td>${{ item.unit_cost.toLocaleString() }}</td>
                <td><strong>${{ item.total_cost.toLocaleString() }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <div class="action-section">
        <button
          @click="submitOrder"
          :disabled="submitting"
          class="submit-button"
        >
          {{ submitting ? 'Placing Order...' : 'Place Restocking Order' }}
        </button>
      </div>

      <div v-if="successMessage" class="success-message">
        {{ successMessage }}
      </div>
    </div>
    <div v-else class="no-recommendations">
      <p>No recommendations available for the selected budget.</p>
      <p>Try increasing your budget or check back later.</p>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const budgetOptions = [10000, 25000, 50000, 100000, 200000]
    const selectedBudget = ref(50000)
    const budgetStep = 5000

    const loading = ref(false)
    const error = ref(null)
    const recommendations = ref([])
    const totalCost = ref(0)
    const budgetRemaining = ref(0)
    const submitting = ref(false)
    const successMessage = ref(null)

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      successMessage.value = null
      try {
        const response = await api.getRestockingRecommendations(selectedBudget.value)
        recommendations.value = response.recommendations || []
        totalCost.value = response.total_cost || 0
        budgetRemaining.value = response.budget_remaining || selectedBudget.value
      } catch (err) {
        error.value = 'Failed to load recommendations: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const handleBudgetChange = () => {
      // Snap to nearest budget option
      const closest = budgetOptions.reduce((prev, curr) => {
        return Math.abs(curr - selectedBudget.value) < Math.abs(prev - selectedBudget.value) ? curr : prev
      })
      selectedBudget.value = closest
    }

    const submitOrder = async () => {
      if (recommendations.value.length === 0) return

      submitting.value = true
      error.value = null

      try {
        const orderData = {
          budget: selectedBudget.value,
          items: recommendations.value.map(item => ({
            item_id: item.item_id,
            sku: item.sku,
            name: item.name,
            quantity: item.quantity_to_order,
            unit_cost: item.unit_cost,
            warehouse: item.warehouse,
            category: item.category
          })),
          total_cost: totalCost.value
        }

        await api.submitRestockingOrder(orderData)
        successMessage.value = 'Restocking order placed successfully! View it in the Orders tab.'

        // Reload recommendations after successful order
        setTimeout(() => {
          loadRecommendations()
        }, 2000)
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    // Watch budget changes and reload recommendations
    watch(selectedBudget, () => {
      loadRecommendations()
    })

    onMounted(() => {
      loadRecommendations()
    })

    return {
      budgetOptions,
      selectedBudget,
      budgetStep,
      loading,
      error,
      recommendations,
      totalCost,
      budgetRemaining,
      submitting,
      successMessage,
      handleBudgetChange,
      submitOrder
    }
  }
}
</script>

<style scoped>
.budget-card {
  margin-bottom: 1.5rem;
}

.budget-display {
  font-size: 2rem;
  font-weight: 700;
  color: #2563eb;
  letter-spacing: -0.025em;
}

.budget-slider-container {
  padding: 1rem 0;
}

.budget-slider {
  width: 100%;
  height: 8px;
  border-radius: 4px;
  background: #e2e8f0;
  outline: none;
  -webkit-appearance: none;
  appearance: none;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 24px;
  height: 24px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  transition: all 0.2s ease;
}

.budget-slider::-webkit-slider-thumb:hover {
  background: #1d4ed8;
  transform: scale(1.1);
}

.budget-slider::-moz-range-thumb {
  width: 24px;
  height: 24px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: none;
  transition: all 0.2s ease;
}

.budget-slider::-moz-range-thumb:hover {
  background: #1d4ed8;
  transform: scale(1.1);
}

.budget-labels {
  display: flex;
  justify-content: space-between;
  margin-top: 0.5rem;
}

.budget-label {
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 500;
}

.budget-summary {
  display: flex;
  gap: 1.5rem;
  align-items: center;
}

.budget-used {
  font-size: 1rem;
  font-weight: 600;
  color: #0f172a;
}

.budget-remaining {
  font-size: 1rem;
  font-weight: 600;
  color: #059669;
}

.restocking-table {
  width: 100%;
  table-layout: auto;
}

.restocking-table th,
.restocking-table td {
  padding: 0.75rem;
}

.action-section {
  display: flex;
  justify-content: center;
  margin-top: 1.5rem;
}

.submit-button {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.875rem 2rem;
  border-radius: 8px;
  font-size: 1rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s ease;
}

.submit-button:hover:not(:disabled) {
  background: #1d4ed8;
  transform: translateY(-1px);
  box-shadow: 0 4px 12px rgba(37, 99, 235, 0.3);
}

.submit-button:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.success-message {
  margin-top: 1.5rem;
  padding: 1rem;
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  border-radius: 8px;
  text-align: center;
  font-weight: 500;
}

.no-recommendations {
  text-align: center;
  padding: 3rem;
  color: #64748b;
}

.no-recommendations p {
  margin-bottom: 0.5rem;
}
</style>
