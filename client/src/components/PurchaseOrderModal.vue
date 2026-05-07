<template>
  <Teleport to="body">
    <Transition name="modal">
      <div v-if="isOpen && backlogItem" class="modal-overlay" @click="close">
        <div class="modal-container" @click.stop>
          <div class="modal-header">
            <h3 class="modal-title">
              {{ mode === 'create' ? 'Create Purchase Order' : 'Purchase Order Details' }}
            </h3>
            <button class="close-button" @click="close">
              <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
                <path d="M15 5L5 15M5 5L15 15" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
              </svg>
            </button>
          </div>

          <div class="modal-body">
            <!-- Item summary header -->
            <div class="item-header">
              <div class="item-info">
                <div class="item-name">{{ backlogItem.item_name }}</div>
                <div class="item-sku">SKU: {{ backlogItem.item_sku }}</div>
              </div>
              <span class="priority-badge" :class="backlogItem.priority?.toLowerCase()">
                {{ backlogItem.priority }} Priority
              </span>
            </div>

            <!-- CREATE MODE: form -->
            <form v-if="mode === 'create'" @submit.prevent="submitForm" class="po-form">
              <div class="shortage-note">
                Shortage: {{ shortageQty }} units
                ({{ backlogItem.quantity_needed }} needed, {{ backlogItem.quantity_available }} available)
              </div>

              <div class="form-group">
                <label class="form-label" for="supplier_name">Supplier Name</label>
                <input
                  id="supplier_name"
                  v-model="form.supplier_name"
                  type="text"
                  class="form-input"
                  placeholder="Enter supplier name"
                  required
                />
              </div>

              <div class="form-row">
                <div class="form-group">
                  <label class="form-label" for="quantity">Quantity</label>
                  <input
                    id="quantity"
                    v-model.number="form.quantity"
                    type="number"
                    class="form-input"
                    min="1"
                    required
                  />
                </div>

                <div class="form-group">
                  <label class="form-label" for="unit_cost">Unit Cost ($)</label>
                  <input
                    id="unit_cost"
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
                <label class="form-label" for="expected_delivery_date">Expected Delivery Date</label>
                <input
                  id="expected_delivery_date"
                  v-model="form.expected_delivery_date"
                  type="date"
                  class="form-input"
                  required
                />
              </div>

              <div class="form-group">
                <label class="form-label" for="notes">Notes <span class="optional-label">(optional)</span></label>
                <textarea
                  id="notes"
                  v-model="form.notes"
                  class="form-textarea"
                  rows="3"
                  placeholder="Add any relevant notes..."
                ></textarea>
              </div>

              <!-- Total cost computed display -->
              <div v-if="form.quantity && form.unit_cost" class="total-cost-preview">
                <span class="total-cost-label">Estimated Total</span>
                <span class="total-cost-value">${{ totalCostDisplay }}</span>
              </div>

              <div v-if="submitError" class="submit-error">{{ submitError }}</div>

              <div class="modal-footer">
                <button type="button" class="btn-secondary" @click="close" :disabled="submitting">Cancel</button>
                <button type="submit" class="btn-primary" :disabled="submitting">
                  {{ submitting ? 'Creating...' : 'Create Purchase Order' }}
                </button>
              </div>
            </form>

            <!-- VIEW MODE: read-only details -->
            <div v-else-if="mode === 'view' && backlogItem.purchase_order" class="po-details">
              <div class="info-grid">
                <div class="info-item">
                  <div class="info-label">PO ID</div>
                  <div class="info-value mono">{{ backlogItem.purchase_order.id }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Status</div>
                  <div class="info-value">
                    <span class="status-badge" :class="statusBadgeClass(backlogItem.purchase_order.status)">
                      {{ backlogItem.purchase_order.status }}
                    </span>
                  </div>
                </div>

                <div class="info-item">
                  <div class="info-label">Supplier</div>
                  <div class="info-value">{{ backlogItem.purchase_order.supplier_name }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Quantity</div>
                  <div class="info-value">{{ backlogItem.purchase_order.quantity }} units</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Unit Cost</div>
                  <div class="info-value">${{ backlogItem.purchase_order.unit_cost?.toFixed(2) }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Total Cost</div>
                  <div class="info-value highlight">
                    ${{ viewTotalCost }}
                  </div>
                </div>

                <div class="info-item">
                  <div class="info-label">Expected Delivery</div>
                  <div class="info-value">{{ formatDate(backlogItem.purchase_order.expected_delivery_date) }}</div>
                </div>

                <div class="info-item">
                  <div class="info-label">Created Date</div>
                  <div class="info-value">{{ formatDate(backlogItem.purchase_order.created_date) }}</div>
                </div>

                <div v-if="backlogItem.purchase_order.notes" class="info-item full-width">
                  <div class="info-label">Notes</div>
                  <div class="info-value notes-text">{{ backlogItem.purchase_order.notes }}</div>
                </div>
              </div>

              <div class="modal-footer">
                <button class="btn-secondary" @click="close">Close</button>
              </div>
            </div>
          </div>
        </div>
      </div>
    </Transition>
  </Teleport>
</template>

<script>
import { ref, computed } from 'vue'
import { api } from '../api'

export default {
  name: 'PurchaseOrderModal',
  props: {
    isOpen: {
      type: Boolean,
      default: false
    },
    backlogItem: {
      type: Object,
      default: null
    },
    mode: {
      type: String,
      default: 'create'
    }
  },
  emits: ['close', 'po-created'],
  setup(props, { emit }) {
    const submitting = ref(false)
    const submitError = ref(null)

    // Shortage qty derived from backlog item
    const shortageQty = computed(() => {
      if (!props.backlogItem) return 0
      return Math.max(0, props.backlogItem.quantity_needed - props.backlogItem.quantity_available)
    })

    // Form state — quantity defaults to the shortage amount
    const form = ref({
      supplier_name: '',
      quantity: shortageQty.value || 1,
      unit_cost: null,
      expected_delivery_date: '',
      notes: ''
    })

    const totalCostDisplay = computed(() => {
      const total = (form.value.quantity || 0) * (form.value.unit_cost || 0)
      return total.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 })
    })

    const viewTotalCost = computed(() => {
      if (!props.backlogItem?.purchase_order) return '0.00'
      const total = (props.backlogItem.purchase_order.quantity || 0) *
                    (props.backlogItem.purchase_order.unit_cost || 0)
      return total.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 })
    })

    const statusBadgeClass = (status) => {
      if (!status) return ''
      const s = status.toLowerCase()
      if (s === 'pending') return 'warning'
      if (s === 'ordered' || s === 'confirmed') return 'info'
      if (s === 'delivered' || s === 'completed') return 'success'
      if (s === 'cancelled') return 'danger'
      return 'default'
    }

    const formatDate = (dateString) => {
      if (!dateString) return 'N/A'
      const date = new Date(dateString)
      if (isNaN(date.getTime())) return dateString
      return date.toLocaleDateString('en-US', {
        year: 'numeric',
        month: 'long',
        day: 'numeric'
      })
    }

    const close = () => {
      emit('close')
    }

    const submitForm = async () => {
      submitting.value = true
      submitError.value = null
      try {
        const payload = {
          backlog_item_id: props.backlogItem.id,
          supplier_name: form.value.supplier_name,
          quantity: form.value.quantity,
          unit_cost: form.value.unit_cost,
          expected_delivery_date: form.value.expected_delivery_date,
          notes: form.value.notes || null
        }
        const created = await api.createPurchaseOrder(payload)
        emit('po-created', created)
      } catch (err) {
        submitError.value = err?.response?.data?.detail || 'Failed to create purchase order. Please try again.'
        console.error('PurchaseOrderModal submitForm error:', err)
      } finally {
        submitting.value = false
      }
    }

    return {
      form,
      submitting,
      submitError,
      shortageQty,
      totalCostDisplay,
      viewTotalCost,
      statusBadgeClass,
      formatDate,
      close,
      submitForm
    }
  }
}
</script>

<style scoped>
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
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
  max-width: 600px;
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
  flex-shrink: 0;
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
  padding: 1.5rem;
}

/* Item summary header inside body */
.item-header {
  display: flex;
  align-items: flex-start;
  justify-content: space-between;
  gap: 1rem;
  padding-bottom: 1.25rem;
  border-bottom: 1px solid #e2e8f0;
  margin-bottom: 1.5rem;
}

.item-info {
  min-width: 0;
}

.item-name {
  font-size: 1rem;
  font-weight: 600;
  color: #0f172a;
  margin-bottom: 0.25rem;
}

.item-sku {
  font-size: 0.813rem;
  color: #64748b;
  font-family: 'Monaco', 'Courier New', monospace;
}

.priority-badge {
  padding: 0.375rem 0.75rem;
  border-radius: 6px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
  flex-shrink: 0;
}

.priority-badge.high {
  background: #fecaca;
  color: #991b1b;
}

.priority-badge.medium {
  background: #fed7aa;
  color: #92400e;
}

.priority-badge.low {
  background: #dbeafe;
  color: #1e40af;
}

/* Shortage note banner */
.shortage-note {
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 8px;
  padding: 0.75rem 1rem;
  font-size: 0.875rem;
  color: #991b1b;
  font-weight: 500;
  margin-bottom: 1.25rem;
}

/* Form */
.po-form {
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.form-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1rem;
}

.form-group {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.form-label {
  font-size: 0.813rem;
  font-weight: 600;
  color: #334155;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

.optional-label {
  font-weight: 400;
  text-transform: none;
  letter-spacing: 0;
  color: #94a3b8;
  font-size: 0.75rem;
}

.form-input {
  padding: 0.5rem 0.75rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.875rem;
  color: #0f172a;
  background: white;
  font-family: inherit;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
  outline: none;
}

.form-input:focus {
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.1);
}

.form-textarea {
  padding: 0.5rem 0.75rem;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  font-size: 0.875rem;
  color: #0f172a;
  background: white;
  font-family: inherit;
  resize: vertical;
  transition: border-color 0.15s ease, box-shadow 0.15s ease;
  outline: none;
}

.form-textarea:focus {
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.1);
}

.total-cost-preview {
  display: flex;
  align-items: center;
  justify-content: space-between;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 0.875rem 1rem;
}

.total-cost-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

.total-cost-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
}

.submit-error {
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 8px;
  padding: 0.75rem 1rem;
  font-size: 0.875rem;
  color: #dc2626;
}

/* View mode details */
.po-details {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
}

.info-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1.25rem;
}

.info-item {
  display: flex;
  flex-direction: column;
  gap: 0.375rem;
}

.info-item.full-width {
  grid-column: 1 / -1;
}

.info-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.info-value {
  font-size: 0.938rem;
  color: #0f172a;
  font-weight: 500;
}

.info-value.mono {
  font-family: 'Monaco', 'Courier New', monospace;
  color: #2563eb;
}

.info-value.highlight {
  font-size: 1.125rem;
  font-weight: 700;
}

.info-value.notes-text {
  color: #475569;
  line-height: 1.5;
}

/* Status badges */
.status-badge {
  display: inline-block;
  padding: 0.25rem 0.625rem;
  border-radius: 4px;
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: capitalize;
}

.status-badge.success {
  background: #dcfce7;
  color: #166534;
}

.status-badge.warning {
  background: #fef9c3;
  color: #854d0e;
}

.status-badge.info {
  background: #dbeafe;
  color: #1e40af;
}

.status-badge.danger {
  background: #fecaca;
  color: #991b1b;
}

.status-badge.default {
  background: #f1f5f9;
  color: #334155;
}

/* Footer */
.modal-footer {
  padding-top: 1.25rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  justify-content: flex-end;
  gap: 0.75rem;
  margin-top: 0.5rem;
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

.btn-secondary:hover:not(:disabled) {
  background: #e2e8f0;
  border-color: #cbd5e1;
}

.btn-secondary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.btn-primary {
  padding: 0.625rem 1.25rem;
  background: #2563eb;
  border: 1px solid #2563eb;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.875rem;
  color: white;
  cursor: pointer;
  transition: all 0.15s ease;
  font-family: inherit;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
  border-color: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

/* Modal transition animations */
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
