<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="isOpen && backlogItem" class="modal-overlay" @click="close">
        <div class="modal-container" @click.stop>
          <div class="modal-header">
            <h3 class="modal-title">{{ mode === 'create' ? 'Create Purchase Order' : 'Purchase Order Details' }}</h3>
            <button class="close-button" @click="close">
              <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
                <path d="M15 5L5 15M5 5L15 15" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
              </svg>
            </button>
          </div>

          <div class="modal-body">
            <div class="item-summary">
              <div class="item-info">
                <span class="item-name">{{ backlogItem.item_name }}</span>
                <span class="item-sku">SKU: {{ backlogItem.item_sku }}</span>
              </div>
              <span class="priority-badge" :class="backlogItem.priority">{{ backlogItem.priority }} Priority</span>
            </div>

            <template v-if="mode === 'create'">
              <div v-if="errorMessage" class="error-message">{{ errorMessage }}</div>
              <form class="po-form" @submit.prevent="submitForm">
                <div class="form-group">
                  <label class="form-label" for="po-supplier">Supplier</label>
                  <input
                    id="po-supplier"
                    v-model="form.supplier"
                    type="text"
                    class="form-input"
                    placeholder="Enter supplier name"
                    required
                  />
                </div>
                <div class="form-row">
                  <div class="form-group">
                    <label class="form-label" for="po-quantity">Quantity</label>
                    <input
                      id="po-quantity"
                      v-model.number="form.quantity"
                      type="number"
                      class="form-input"
                      min="1"
                      required
                    />
                  </div>
                  <div class="form-group">
                    <label class="form-label" for="po-unit-cost">Unit Cost ($)</label>
                    <input
                      id="po-unit-cost"
                      v-model.number="form.unit_cost"
                      type="number"
                      class="form-input"
                      min="0"
                      step="0.01"
                      placeholder="0.00"
                      required
                    />
                  </div>
                </div>
                <div class="form-group">
                  <label class="form-label" for="po-delivery">Expected Delivery</label>
                  <input
                    id="po-delivery"
                    v-model="form.expected_delivery"
                    type="date"
                    class="form-input"
                    required
                  />
                </div>
              </form>
            </template>

            <template v-else>
              <template v-if="backlogItem.purchase_order">
                <div class="info-grid">
                  <div class="info-item">
                    <div class="info-label">Supplier</div>
                    <div class="info-value">{{ backlogItem.purchase_order.supplier }}</div>
                  </div>
                  <div class="info-item">
                    <div class="info-label">Quantity</div>
                    <div class="info-value">{{ backlogItem.purchase_order.quantity }} units</div>
                  </div>
                  <div class="info-item">
                    <div class="info-label">Unit Cost</div>
                    <div class="info-value">{{ formatCurrency(backlogItem.purchase_order.unit_cost) }}</div>
                  </div>
                  <div class="info-item">
                    <div class="info-label">Status</div>
                    <div class="info-value">
                      <span class="badge info">{{ backlogItem.purchase_order.status || 'Pending' }}</span>
                    </div>
                  </div>
                  <div class="info-item">
                    <div class="info-label">Expected Delivery</div>
                    <div class="info-value">{{ formatDate(backlogItem.purchase_order.expected_delivery) }}</div>
                  </div>
                  <div class="info-item">
                    <div class="info-label">Created</div>
                    <div class="info-value">{{ formatDate(backlogItem.purchase_order.created_at) }}</div>
                  </div>
                </div>
              </template>
              <div v-else class="empty-state">No purchase order found for this item.</div>
            </template>
          </div>

          <div class="modal-footer">
            <button class="btn-secondary" @click="close">{{ mode === 'create' ? 'Cancel' : 'Close' }}</button>
            <button
              v-if="mode === 'create'"
              class="btn-primary"
              :disabled="submitting"
              @click="submitForm"
            >
              {{ submitting ? 'Creating...' : 'Create PO' }}
            </button>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script setup>
import { ref, watch } from 'vue'
import { api } from '../api'

const props = defineProps({
  isOpen: { type: Boolean, default: false },
  backlogItem: { type: Object, default: null },
  mode: { type: String, default: 'create' }
})

const emit = defineEmits(['close', 'po-created'])

const submitting = ref(false)
const errorMessage = ref('')
const form = ref({ supplier: '', quantity: 0, unit_cost: '', expected_delivery: '' })

watch(() => props.isOpen, (open) => {
  if (open && props.mode === 'create') {
    form.value = {
      supplier: '',
      quantity: props.backlogItem?.quantity_needed ?? 1,
      unit_cost: '',
      expected_delivery: ''
    }
    errorMessage.value = ''
  }
})

const close = () => emit('close')

const submitForm = async () => {
  submitting.value = true
  errorMessage.value = ''
  try {
    const poData = await api.createPurchaseOrder({
      backlog_item_id: props.backlogItem.id,
      supplier: form.value.supplier,
      quantity: form.value.quantity,
      unit_cost: form.value.unit_cost,
      expected_delivery: form.value.expected_delivery
    })
    emit('po-created', poData)
    emit('close')
  } catch (err) {
    errorMessage.value = 'Failed to create purchase order. Please try again.'
  } finally {
    submitting.value = false
  }
}

const formatCurrency = (value) => {
  if (value == null) return 'N/A'
  return Number(value).toLocaleString('en-US', { style: 'currency', currency: 'USD' })
}

const formatDate = (dateString) => {
  if (!dateString) return 'N/A'
  const date = new Date(dateString)
  return isNaN(date.getTime()) ? 'N/A' : date.toLocaleDateString('en-US', { year: 'numeric', month: 'long', day: 'numeric' })
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0; left: 0; right: 0; bottom: 0;
  background: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
  padding: 1rem;
}

.modal-container {
  background: white;
  border-radius: 12px;
  box-shadow: 0 20px 50px rgba(0, 0, 0, 0.15);
  max-width: 560px;
  width: 100%;
  max-height: 90vh;
  overflow: hidden;
  display: flex;
  flex-direction: column;
}

.modal-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1.5rem;
  border-bottom: 1px solid #e2e8f0;
}

.modal-title {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.close-button {
  background: none;
  border: none;
  color: #64748b;
  cursor: pointer;
  padding: 0.5rem;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 6px;
  transition: all 0.15s ease;
}

.close-button:hover {
  background: #f1f5f9;
  color: #0f172a;
}

.modal-body {
  flex: 1;
  overflow-y: auto;
  padding: 1.5rem 2rem;
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

.item-summary {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  padding: 1rem 1.25rem;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
}

.item-info {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
  min-width: 0;
}

.item-name {
  font-weight: 600;
  font-size: 0.9375rem;
  color: #0f172a;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.item-sku {
  font-size: 0.8125rem;
  color: #64748b;
  font-family: 'Monaco', 'Courier New', monospace;
}

.priority-badge {
  padding: 0.375rem 0.875rem;
  border-radius: 6px;
  font-size: 0.8125rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
  flex-shrink: 0;
}

.priority-badge.high { background: #fecaca; color: #991b1b; }
.priority-badge.medium { background: #fed7aa; color: #92400e; }
.priority-badge.low { background: #dbeafe; color: #1e40af; }

.error-message {
  padding: 0.75rem 1rem;
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 8px;
  color: #dc2626;
  font-size: 0.875rem;
}

.po-form {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.form-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.form-label {
  font-size: 0.8125rem;
  font-weight: 600;
  color: #334155;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

.form-input {
  padding: 0.625rem 0.875rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.9375rem;
  color: #0f172a;
  font-family: inherit;
  outline: none;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
  background: white;
  width: 100%;
}

.form-input:focus {
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.info-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 1.25rem;
}

.info-item {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.info-label {
  font-size: 0.8125rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.info-value {
  font-size: 0.9375rem;
  color: #0f172a;
  font-weight: 500;
}

.badge {
  display: inline-flex;
  align-items: center;
  padding: 3px 10px;
  border-radius: 999px;
  font-size: 0.6875rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

.badge.info { background: #dbeafe; color: #2563eb; }

.empty-state {
  text-align: center;
  color: #64748b;
  font-size: 0.9375rem;
  padding: 2rem 0;
}

.modal-footer {
  padding: 1.5rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
}

.btn-secondary {
  padding: 0.625rem 1.25rem;
  background: #f1f5f9;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-weight: 500;
  font-size: 0.875rem;
  color: #334155;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-secondary:hover {
  background: #e2e8f0;
  border-color: #cbd5e1;
}

.btn-primary {
  padding: 0.625rem 1.25rem;
  background: #3b82f6;
  border: 1px solid #3b82f6;
  border-radius: 8px;
  font-weight: 500;
  font-size: 0.875rem;
  color: white;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-primary:hover:not(:disabled) {
  background: #2563eb;
  border-color: #2563eb;
}

.btn-primary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.modal-enter-active,
.modal-leave-active {
  transition: opacity 0.2s ease;
}

.modal-enter-from,
.modal-leave-to {
  opacity: 0;
}

.modal-enter-active .modal-container,
.modal-leave-active .modal-container {
  transition: transform 0.2s ease;
}

.modal-enter-from .modal-container,
.modal-leave-to .modal-container {
  transform: scale(0.95);
}
</style>
