<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Set your available budget to get recommended restocking items based on demand forecasts.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Card -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">Available Budget</h3>
        </div>
        <div class="slider-row">
          <input
            type="range"
            min="0"
            max="100000"
            step="1000"
            v-model.number="budget"
          />
          <span class="budget-display">${{ budget.toLocaleString() }}</span>
        </div>
        <p class="budget-note">Showing {{ recommendedItems.length }} items within budget</p>
      </div>

      <!-- Success Banner -->
      <div v-if="orderPlaced" class="success-banner">
        Restocking order placed successfully! View it in the Orders tab.
      </div>

      <!-- Recommendations Table Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items ({{ recommendedItems.length }})</h3>
          <button
            class="place-order-btn"
            @click="placeOrder"
            :disabled="recommendedItems.length === 0 || orderPlaced || placingOrder"
          >
            {{ placingOrder ? 'Placing...' : orderPlaced ? 'Order Placed' : 'Place Order' }}
          </button>
        </div>
        <div v-if="recommendedItems.length === 0" class="empty-state">
          No items fit within the current budget. Try increasing the budget.
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>Item Name</th>
                <th>SKU</th>
                <th>Qty (Forecasted Demand)</th>
                <th>Unit Cost</th>
                <th>Total Cost</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.sku">
                <td>{{ item.item_name }}</td>
                <td>{{ item.sku }}</td>
                <td>{{ item.quantity }}</td>
                <td>${{ item.unit_cost.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</td>
                <td>${{ item.total_cost.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</td>
              </tr>
            </tbody>
            <tfoot>
              <tr class="total-row">
                <td colspan="4"><strong>Total Spend</strong></td>
                <td :class="{ 'over-budget': totalSpend >= budget }">
                  <strong>${{ totalSpend.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</strong>
                  <span> / ${{ budget.toLocaleString() }}</span>
                </td>
              </tr>
            </tfoot>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const budget = ref(50000)
    const loading = ref(true)
    const error = ref(null)
    const demandItems = ref([])
    const inventoryItems = ref([])
    const orderPlaced = ref(false)
    const placingOrder = ref(false)

    const recommendedItems = computed(() => {
      // Build a map of inventory by SKU for O(1) lookup
      const inventoryBySku = {}
      inventoryItems.value.forEach(inv => {
        inventoryBySku[inv.sku] = { unit_cost: inv.unit_cost }
      })

      // For each demand item, find its unit_cost and compute item_total
      const candidates = []
      demandItems.value.forEach(item => {
        const invEntry = inventoryBySku[item.item_sku]
        if (!invEntry || !invEntry.unit_cost) return

        const unit_cost = invEntry.unit_cost
        const item_total = item.forecasted_demand * unit_cost
        candidates.push({
          sku: item.item_sku,
          item_name: item.item_name,
          quantity: item.forecasted_demand,
          unit_cost,
          total_cost: item_total
        })
      })

      // Sort by forecasted_demand descending
      candidates.sort((a, b) => b.quantity - a.quantity)

      // Greedy fill within budget
      let runningTotal = 0
      const result = []
      for (const item of candidates) {
        if (runningTotal + item.total_cost <= budget.value) {
          runningTotal += item.total_cost
          result.push(item)
        }
      }

      return result
    })

    const totalSpend = computed(() => {
      return recommendedItems.value.reduce((sum, item) => sum + item.total_cost, 0)
    })

    const placeOrder = async () => {
      placingOrder.value = true
      try {
        await api.createRestockingOrder({
          items: recommendedItems.value.map(item => ({
            sku: item.sku,
            item_name: item.item_name,
            quantity: item.quantity,
            unit_cost: item.unit_cost,
            total_cost: item.total_cost
          })),
          total_value: totalSpend.value
        })
        orderPlaced.value = true
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        placingOrder.value = false
      }
    }

    onMounted(async () => {
      try {
        loading.value = true
        error.value = null
        const [demand, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        demandItems.value = demand
        inventoryItems.value = inventory
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
      } finally {
        loading.value = false
      }
    })

    return {
      budget,
      loading,
      error,
      demandItems,
      inventoryItems,
      orderPlaced,
      placingOrder,
      recommendedItems,
      totalSpend,
      placeOrder
    }
  }
}
</script>

<style scoped>
.slider-row {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-bottom: 0.75rem;
}

input[type="range"] {
  width: 100%;
  accent-color: #3b82f6;
}

.budget-display {
  font-size: 1.5rem;
  font-weight: 700;
  color: #f8fafc;
  white-space: nowrap;
}

.budget-note {
  font-size: 0.813rem;
  color: #64748b;
}

.place-order-btn {
  background: #3b82f6;
  color: white;
  padding: 0.5rem 1.25rem;
  border-radius: 6px;
  border: none;
  cursor: pointer;
  font-weight: 600;
  font-size: 0.875rem;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #2563eb;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.success-banner {
  background: #14532d;
  color: #4ade80;
  padding: 1rem 1.5rem;
  border-radius: 8px;
  margin-bottom: 1rem;
  border: 1px solid #166534;
  font-weight: 500;
}

.empty-state {
  text-align: center;
  padding: 2rem;
  color: #64748b;
}

.total-row {
  font-weight: 700;
  background: #0f172a;
  border-top: 2px solid #334155;
}

.total-row td {
  color: #e2e8f0;
}

.over-budget {
  color: #ef4444;
}
</style>
