<template>
  <div v-if="visible" class="card-payment-overlay">
    <div class="card-payment-container">
      <!-- 头部 -->
      <div class="payment-header">
        <h1 class="payment-title">Complete payment</h1>
        <button class="close-btn" @click="handleCancel" aria-label="Close">
          <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
            <path d="M15 5L5 15M5 5l10 10" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
          </svg>
        </button>
      </div>

      <!-- 订单摘要 -->
      <div class="order-summary-section">
        <div class="summary-item">
          <span class="item-label">Item</span>
          <span class="item-value">{{ productName }}</span>
        </div>
        <div class="summary-item">
          <span class="item-label">Amount</span>
          <span class="item-value amount">{{ formatCurrency(productPrice, currency) }}</span>
        </div>
      </div>

      <!-- 卡片信息显示 -->
      <div class="card-display-section">
        <div class="card-preview">
          <div class="card-chip"></div>
          <div class="card-number-display">{{ maskedCardNumber }}</div>
          <div class="card-footer">
            <div class="card-holder-name">{{ cardForm.name || 'CARDHOLDER NAME' }}</div>
            <div class="card-expiry-display">{{ cardForm.expiry || 'MM/YY' }}</div>
          </div>
        </div>
        <!-- 3DS验证提示（仅在3DS场景显示） -->
        <div v-if="props.paymentScenario === '3ds_verification' && currentStep === 'verification'" class="3ds-notice">
          <div class="notice-icon">🔒</div>
          <p class="notice-text">This payment requires 3D Secure authentication</p>
        </div>
      </div>

      <!-- 3D Secure验证（仅在3DS验证场景显示） -->
      <div v-if="currentStep === 'verification' && props.paymentScenario === '3ds_verification'" class="verification-section">
        <div class="verification-header">
          <div class="bank-logo">
            <svg width="48" height="48" viewBox="0 0 48 48" fill="none">
              <rect x="8" y="12" width="32" height="24" rx="4" stroke="currentColor" stroke-width="2" fill="#f9fafb"/>
              <path d="M16 20h16M16 24h16M16 28h12" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
            </svg>
          </div>
          <h2 class="verification-title">Complete authentication</h2>
          <p class="verification-description">Your bank requires additional authentication to complete this payment.</p>
        </div>

        <!-- 银行验证界面（模拟 iframe） -->
        <div class="bank-challenge-container">
          <div class="bank-challenge-header">
            <div class="bank-name">{{ bankName }}</div>
            <div class="challenge-secure-badge">
              <svg width="12" height="12" viewBox="0 0 12 12" fill="none">
                <path d="M6 1L3 2.5v3c0 2.5 1.5 4.5 3 5.5 1.5-1 3-3 3-5.5v-3L6 1z" fill="currentColor"/>
              </svg>
              <span>Secure</span>
            </div>
          </div>
          
          <div class="challenge-content">
            <div class="challenge-message">
              <p>Please enter your password to authenticate this payment</p>
              <div class="challenge-amount">
                <span class="amount-label">Amount:</span>
                <span class="amount-value">{{ formatCurrency(productPrice, currency) }}</span>
              </div>
            </div>
            
            <div class="challenge-form">
              <label class="challenge-label">Password</label>
              <input 
                type="password" 
                v-model="challengePassword" 
                @input="onChallengeInput"
                placeholder="Enter your password"
                class="challenge-input"
                :disabled="isProcessing"
                autofocus
              />
              <div v-if="challengeError" class="challenge-error">{{ challengeError }}</div>
            </div>
            
            <div class="challenge-actions">
              <button 
                class="challenge-submit-btn" 
                @click="handleChallengeSubmit"
                :disabled="!challengePassword || challengePassword.length < 4 || isProcessing"
              >
                {{ isProcessing ? 'Authenticating...' : 'Continue' }}
              </button>
              <button 
                class="challenge-cancel-btn" 
                @click="handleCancel"
                :disabled="isProcessing"
              >
                Cancel
              </button>
            </div>
          </div>
        </div>
      </div>

      <!-- 支付处理中 -->
      <div v-if="currentStep === 'processing'" class="processing-section">
        <div class="processing-spinner-large">
          <svg width="48" height="48" viewBox="0 0 48 48">
            <circle cx="24" cy="24" r="20" stroke="#e5e7eb" stroke-width="4" fill="none"/>
            <circle cx="24" cy="24" r="20" stroke="#111827" stroke-width="4" fill="none" 
                    stroke-dasharray="125" stroke-dashoffset="31" stroke-linecap="round" class="spinner-circle"/>
          </svg>
        </div>
        <h2 class="processing-title">Processing your payment</h2>
        <p class="processing-description">Please wait, don't close this window</p>
      </div>

      <!-- 支付成功 -->
      <div v-if="currentStep === 'success'" class="success-section">
        <div class="success-icon">
          <svg width="64" height="64" viewBox="0 0 64 64" fill="none">
            <circle cx="32" cy="32" r="30" fill="#10b981"/>
            <path d="M20 32l8 8 16-16" stroke="white" stroke-width="4" stroke-linecap="round" stroke-linejoin="round"/>
          </svg>
        </div>
        <h2 class="success-title">Payment successful</h2>
        <p class="success-description">Your payment has been processed successfully</p>
        <div class="success-details">
          <div class="detail-row">
            <span class="detail-label">Order ID</span>
            <span class="detail-value">{{ orderId }}</span>
          </div>
          <div class="detail-row">
            <span class="detail-label">Amount</span>
            <span class="detail-value">{{ formatCurrency(productPrice, currency) }}</span>
          </div>
          <div class="detail-row">
            <span class="detail-label">Payment method</span>
            <span class="detail-value">Card ending {{ last4Digits }}</span>
          </div>
        </div>
        <button class="done-button" @click="handleReturn">Done</button>
      </div>

      <!-- 支付失败 -->
      <div v-if="currentStep === 'error'" class="error-section">
        <div class="error-icon">
          <svg width="64" height="64" viewBox="0 0 64 64" fill="none">
            <circle cx="32" cy="32" r="30" fill="#ef4444"/>
            <path d="M32 20v24M32 44h.01" stroke="white" stroke-width="4" stroke-linecap="round"/>
          </svg>
        </div>
        <h2 class="error-title">Payment failed</h2>
        <p class="error-description">{{ errorMessage }}</p>
        <div class="error-actions">
          <button class="retry-button" @click="handleRetry">Try again</button>
          <button class="cancel-button" @click="handleReturn">Cancel</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch } from 'vue'

const props = defineProps({
  visible: {
    type: Boolean,
    default: false
  },
  productName: {
    type: String,
    default: 'Game Item'
  },
  productPrice: {
    type: String,
    default: '9.99'
  },
  paymentScenario: {
    type: String,
    default: 'normal'
  },
  cardForm: {
    type: Object,
    default: () => ({
      name: '',
      number: '',
      expiry: '',
      cvv: ''
    })
  },
  currency: {
    type: String,
    default: 'USD'
  }
})

const emit = defineEmits(['close', 'payment-success', 'payment-error'])

// 根据paymentScenario决定初始步骤
const getInitialStep = () => {
  // 只有3ds_verification场景才显示3DS验证，其他场景直接进入processing
  return props.paymentScenario === '3ds_verification' ? 'verification' : 'processing'
}

const currentStep = ref(getInitialStep()) // 'verification', 'processing', 'success', 'error'
const challengePassword = ref('')
const challengeError = ref('')
const orderId = ref('')
const errorMessage = ref('')
const bankName = ref('Your Bank')

const isProcessing = computed(() => currentStep.value === 'processing')

const maskedCardNumber = computed(() => {
  const number = props.cardForm.number.replace(/\s/g, '')
  if (number.length >= 4) {
    const last4 = number.slice(-4)
    return '•••• •••• •••• ' + last4
  }
  return '•••• •••• •••• ••••'
})

const last4Digits = computed(() => {
  const number = props.cardForm.number.replace(/\s/g, '')
  return number.slice(-4) || '••••'
})

const formatCurrency = (amount, currency) => {
  const symbols = {
    USD: '$',
    CNY: '¥',
    EUR: '€',
    GBP: '£',
    JPY: '¥'
  }
  return `${symbols[currency] || currency}${parseFloat(amount).toFixed(2)}`
}

const onChallengeInput = () => {
  challengeError.value = ''
}

const handleChallengeSubmit = async () => {
  if (!challengePassword.value || challengePassword.value.length < 4 || isProcessing.value) return
  
  // 验证密码（模拟）
  if (challengePassword.value.length < 6) {
    challengeError.value = 'Password is too short'
    return
  }
  
  currentStep.value = 'processing'
  
  try {
    // 模拟银行验证
    await simulate3DSChallenge()
    
    // 3DS验证成功后，继续支付流程
    await simulatePayment()
    
    if (props.paymentScenario === '3ds_verification' || props.paymentScenario === 'normal') {
      currentStep.value = 'success'
      orderId.value = 'ORD' + Date.now().toString().slice(-10)
      setTimeout(() => {
        emit('payment-success')
        setTimeout(() => {
          handleReturn()
        }, 7000)
      }, 3000)
    } else {
      handlePaymentError()
    }
  } catch (error) {
    // 3DS验证失败，返回验证页面
    currentStep.value = 'verification'
    challengeError.value = error.message || 'Authentication failed. Please try again.'
    challengePassword.value = ''
  }
}

const simulate3DSChallenge = () => {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      // 模拟验证失败（如果密码是 'error'）
      if (challengePassword.value === 'error') {
        reject(new Error('Authentication failed'))
      } else {
        resolve()
      }
    }, 1500)
  })
}

// 直接支付（跳过3DS验证）
const handleDirectPayment = async () => {
  if (isProcessing.value) return
  
  currentStep.value = 'processing'
  
  try {
    await simulatePayment()
    
    if (props.paymentScenario === 'normal') {
      currentStep.value = 'success'
      orderId.value = 'ORD' + Date.now().toString().slice(-10)
      setTimeout(() => {
        emit('payment-success')
        setTimeout(() => {
          handleReturn()
        }, 7000)
      }, 3000)
    } else {
      handlePaymentError()
    }
  } catch (error) {
    handlePaymentError()
  }
}

const simulatePayment = () => {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (props.paymentScenario === 'timeout') {
        reject(new Error('Payment timeout'))
      } else {
        resolve()
      }
    }, 2500)
  })
}

const handlePaymentError = () => {
  currentStep.value = 'error'
  
  switch (props.paymentScenario) {
    case 'card_error':
      errorMessage.value = 'Your card was declined. Please check your card details or try another card.'
      break
    case 'network_error':
      errorMessage.value = 'Network error. Please check your connection and try again.'
      break
    case 'timeout':
      errorMessage.value = 'Payment timeout. Please try again.'
      break
    default:
      errorMessage.value = 'Payment failed. Please try again.'
  }
  
  setTimeout(() => {
    emit('payment-error', errorMessage.value)
  }, 3000)
}

const handleRetry = () => {
  currentStep.value = 'verification'
  challengePassword.value = ''
  challengeError.value = ''
}

const handleCancel = () => {
  if (!isProcessing.value) {
    emit('close')
  }
}

const handleReturn = () => {
  emit('close')
}

// 监听visible和paymentScenario变化，重置状态
watch(() => props.visible, (newVal) => {
  if (newVal) {
    resetState()
  }
})

watch(() => props.paymentScenario, () => {
  if (props.visible) {
    resetState()
  }
})

const resetState = () => {
  // 根据paymentScenario决定初始步骤
  currentStep.value = getInitialStep()
  challengePassword.value = ''
  challengeError.value = ''
  orderId.value = ''
  errorMessage.value = ''
  
  // 如果不是3DS验证场景，自动开始支付流程
  if (props.paymentScenario !== '3ds_verification') {
    // 延迟一下，让用户看到卡片信息
    setTimeout(() => {
      handleDirectPayment()
    }, 500)
  }
}
</script>

<style scoped>
/* Adyen/Stripe风格 - 专业简洁 */
.card-payment-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.6);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
  padding: 20px;
  animation: fadeIn 0.2s ease;
}

.card-payment-container {
  background: #ffffff;
  border-radius: 8px;
  width: 100%;
  max-width: 480px;
  max-height: 90vh;
  display: flex;
  flex-direction: column;
  box-shadow: 0 4px 24px rgba(0, 0, 0, 0.15);
  animation: slideUp 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  overflow: hidden;
}

.payment-header {
  padding: 24px 24px 20px;
  border-bottom: 1px solid #e5e7eb;
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
}

.payment-title {
  font-size: 20px;
  font-weight: 600;
  color: #111827;
  margin: 0;
  line-height: 1.4;
}

.close-btn {
  background: none;
  border: none;
  padding: 4px;
  cursor: pointer;
  color: #6b7280;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: color 0.15s;
  margin-left: 16px;
}

.close-btn:hover {
  color: #111827;
}

/* 订单摘要 */
.order-summary-section {
  padding: 20px 24px;
  background-color: #f9fafb;
  border-bottom: 1px solid #e5e7eb;
}

.summary-item {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.summary-item:last-child {
  margin-bottom: 0;
}

.item-label {
  font-size: 14px;
  color: #6b7280;
}

.item-value {
  font-size: 14px;
  color: #111827;
  font-weight: 500;
}

.item-value.amount {
  font-size: 18px;
  font-weight: 600;
}

/* 卡片预览 */
.card-display-section {
  padding: 24px;
  border-bottom: 1px solid #e5e7eb;
}

.card-preview {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border-radius: 12px;
  padding: 24px;
  color: #ffffff;
  min-height: 200px;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
  position: relative;
  overflow: hidden;
}

.card-preview::before {
  content: '';
  position: absolute;
  top: -50%;
  right: -50%;
  width: 200%;
  height: 200%;
  background: radial-gradient(circle, rgba(255,255,255,0.1) 0%, transparent 70%);
}

.card-chip {
  width: 48px;
  height: 36px;
  background: linear-gradient(135deg, #ffd700 0%, #ffed4e 100%);
  border-radius: 6px;
  position: relative;
  margin-bottom: 24px;
}

.card-chip::after {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 32px;
  height: 24px;
  border: 1px solid rgba(0,0,0,0.2);
  border-radius: 4px;
}

.card-number-display {
  font-size: 24px;
  font-weight: 600;
  letter-spacing: 2px;
  font-family: 'Monaco', 'Menlo', monospace;
  margin-bottom: 24px;
}

.card-footer {
  display: flex;
  justify-content: space-between;
  align-items: flex-end;
}

.card-holder-name {
  font-size: 14px;
  text-transform: uppercase;
  letter-spacing: 1px;
  opacity: 0.9;
}

.card-expiry-display {
  font-size: 14px;
  font-weight: 500;
  opacity: 0.9;
}

.3ds-notice {
  margin-top: 16px;
  padding: 12px 16px;
  background-color: #fef3c7;
  border: 1px solid #fde68a;
  border-radius: 6px;
  display: flex;
  align-items: center;
  gap: 8px;
}

.notice-icon {
  font-size: 20px;
  flex-shrink: 0;
}

.notice-text {
  font-size: 13px;
  color: #92400e;
  margin: 0;
}

/* 验证部分 */
.verification-section {
  padding: 24px;
  flex: 1;
  overflow-y: auto;
}

.verification-header {
  text-align: center;
  margin-bottom: 24px;
}

.verification-title {
  font-size: 20px;
  font-weight: 600;
  color: #111827;
  margin: 0 0 8px 0;
}

.verification-description {
  font-size: 14px;
  color: #6b7280;
  margin: 0;
  line-height: 1.5;
}

/* 银行验证界面（模拟 iframe） */
.bank-challenge-container {
  background: #ffffff;
  border: 1px solid #e5e7eb;
  border-radius: 8px;
  overflow: hidden;
  margin-top: 24px;
}

.bank-challenge-header {
  padding: 16px 20px;
  background: #f9fafb;
  border-bottom: 1px solid #e5e7eb;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.bank-name {
  font-size: 15px;
  font-weight: 600;
  color: #111827;
}

.challenge-secure-badge {
  display: flex;
  align-items: center;
  gap: 4px;
  font-size: 12px;
  color: #10b981;
  font-weight: 500;
}

.challenge-secure-badge svg {
  color: #10b981;
}

.challenge-content {
  padding: 24px;
}

.challenge-message {
  margin-bottom: 24px;
}

.challenge-message p {
  font-size: 14px;
  color: #374151;
  margin: 0 0 16px 0;
  line-height: 1.5;
}

.challenge-amount {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 12px 16px;
  background: #f9fafb;
  border-radius: 6px;
}

.amount-label {
  font-size: 14px;
  color: #6b7280;
}

.amount-value {
  font-size: 16px;
  font-weight: 600;
  color: #111827;
}

.challenge-form {
  margin-bottom: 24px;
}

.challenge-label {
  display: block;
  font-size: 13px;
  font-weight: 500;
  color: #374151;
  margin-bottom: 8px;
}

.challenge-input {
  width: 100%;
  padding: 12px 16px;
  border: 1.5px solid #d1d5db;
  border-radius: 6px;
  font-size: 15px;
  color: #111827;
  transition: all 0.15s;
  font-family: inherit;
}

.challenge-input:focus {
  outline: none;
  border-color: #111827;
  box-shadow: 0 0 0 3px rgba(17, 24, 39, 0.1);
}

.challenge-input:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.challenge-input::placeholder {
  color: #9ca3af;
}

.challenge-error {
  margin-top: 8px;
  font-size: 13px;
  color: #ef4444;
}

.challenge-actions {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.challenge-submit-btn {
  width: 100%;
  padding: 14px 24px;
  background-color: #111827;
  color: #ffffff;
  border: none;
  border-radius: 6px;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.15s;
}

.challenge-submit-btn:hover:not(:disabled) {
  background-color: #1f2937;
}

.challenge-submit-btn:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.challenge-cancel-btn {
  width: 100%;
  padding: 12px 24px;
  background: transparent;
  color: #6b7280;
  border: none;
  font-size: 14px;
  cursor: pointer;
  transition: color 0.15s;
}

.challenge-cancel-btn:hover:not(:disabled) {
  color: #111827;
}

.challenge-cancel-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.bank-logo {
  color: #111827;
  margin-bottom: 16px;
}

/* 处理中 */
.processing-section {
  padding: 60px 24px;
  text-align: center;
}

.processing-spinner-large {
  margin-bottom: 24px;
  display: flex;
  justify-content: center;
}

.spinner-circle {
  animation: spin 1s linear infinite;
  transform-origin: center;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.processing-title {
  font-size: 20px;
  font-weight: 600;
  color: #111827;
  margin: 0 0 8px 0;
}

.processing-description {
  font-size: 14px;
  color: #6b7280;
  margin: 0;
}

/* 成功 */
.success-section {
  padding: 40px 24px;
  text-align: center;
}

.success-icon {
  margin-bottom: 24px;
}

.success-title {
  font-size: 20px;
  font-weight: 600;
  color: #111827;
  margin: 0 0 8px 0;
}

.success-description {
  font-size: 14px;
  color: #6b7280;
  margin: 0 0 24px 0;
}

.success-details {
  background-color: #f9fafb;
  border-radius: 8px;
  padding: 20px;
  margin-bottom: 24px;
  text-align: left;
}

.detail-row {
  display: flex;
  justify-content: space-between;
  margin-bottom: 12px;
}

.detail-row:last-child {
  margin-bottom: 0;
}

.detail-label {
  font-size: 14px;
  color: #6b7280;
}

.detail-value {
  font-size: 14px;
  color: #111827;
  font-weight: 500;
}

.done-button {
  width: 100%;
  padding: 14px 24px;
  background-color: #111827;
  color: #ffffff;
  border: none;
  border-radius: 6px;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.15s;
}

.done-button:hover {
  background-color: #1f2937;
}

/* 错误 */
.error-section {
  padding: 40px 24px;
  text-align: center;
}

.error-icon {
  margin-bottom: 24px;
}

.error-title {
  font-size: 20px;
  font-weight: 600;
  color: #111827;
  margin: 0 0 8px 0;
}

.error-description {
  font-size: 14px;
  color: #6b7280;
  margin: 0 0 24px 0;
}

.error-actions {
  display: flex;
  gap: 12px;
}

.retry-button,
.cancel-button {
  flex: 1;
  padding: 12px 24px;
  border: none;
  border-radius: 6px;
  font-size: 15px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.15s;
}

.retry-button {
  background-color: #111827;
  color: #ffffff;
}

.retry-button:hover {
  background-color: #1f2937;
}

.cancel-button {
  background-color: #f3f4f6;
  color: #111827;
}

.cancel-button:hover {
  background-color: #e5e7eb;
}

.field-label {
  display: block;
  font-size: 13px;
  font-weight: 500;
  color: #374151;
  margin-bottom: 8px;
}

/* 响应式 */
@media (max-width: 640px) {
  .card-payment-container {
    max-width: 100%;
    border-radius: 0;
    max-height: 100vh;
  }

  .payment-header {
    padding: 20px 20px 16px;
  }

  .verification-section {
    padding: 20px;
  }

  .code-input-wrapper {
    flex-direction: column;
  }

  .resend-btn {
    width: 100%;
  }
}
</style>

