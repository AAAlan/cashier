<template>
  <div v-if="visible" class="checkout-overlay" @click.self="handleOverlayClick">
    <div class="checkout-container">
      <!-- 头部 -->
      <div class="checkout-header">
        <div class="header-content">
          <h1 class="checkout-title">Complete your payment</h1>
          <p class="checkout-subtitle">Secure payment powered by Cashier</p>
        </div>
        <button class="close-btn" @click="handleClose" aria-label="Close">
          <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
            <path d="M15 5L5 15M5 5l10 10" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
          </svg>
        </button>
      </div>

      <!-- 订单摘要 -->
      <div class="order-summary">
        <div class="summary-row">
          <span class="summary-label">Item</span>
          <span class="summary-value">{{ productName }}</span>
        </div>
        <div class="summary-row">
          <span class="summary-label">Amount</span>
          <span class="summary-value summary-amount">
            {{ formatCurrency(productPrice, currentCurrency) }}
          </span>
        </div>
      </div>

      <!-- 支付方式选择 -->
      <div class="payment-section">
        <h2 class="section-title">Payment method</h2>
        
        <!-- 支付方式列表 -->
        <div class="payment-methods-list">
          <div 
            v-for="method in availablePaymentMethods" 
            :key="method.id"
            class="payment-method-item"
            :class="{ 
              active: paymentMethod === method.id,
              disabled: !method.enabled 
            }"
            @click="selectPaymentMethod(method.id)"
          >
            <div class="method-radio">
              <div class="radio-circle" :class="{ checked: paymentMethod === method.id }"></div>
            </div>
            <div class="method-content">
              <div class="method-header">
                <span class="method-name">{{ method.name }}</span>
                <span v-if="method.badge" class="method-badge">{{ method.badge }}</span>
              </div>
              <div v-if="method.description" class="method-description">{{ method.description }}</div>
            </div>
            <div class="method-icon-wrapper">
              <div class="method-icon">{{ method.icon }}</div>
            </div>
          </div>
        </div>

        <!-- 银行卡支付表单 -->
        <div v-if="paymentMethod === 'card'" class="card-form-container">
          <div class="form-field">
            <label class="field-label">Card number</label>
            <div class="input-wrapper" :class="{ focused: cardNumberFocused, error: cardNumberError }">
              <input 
                type="text" 
                v-model="cardForm.number" 
                @input="formatCardNumber"
                @focus="cardNumberFocused = true"
                @blur="cardNumberFocused = false; validateCardNumber()"
                placeholder="1234 5678 9012 3456"
                maxlength="19"
                class="card-input"
                :disabled="isProcessing"
              />
              <div class="card-brands">
                <span v-if="detectedCardBrand" class="card-brand">{{ detectedCardBrand }}</span>
              </div>
            </div>
            <div v-if="cardNumberError" class="field-error">{{ cardNumberError }}</div>
          </div>

          <div class="form-field">
            <label class="field-label">Cardholder name</label>
            <input 
              type="text" 
              v-model="cardForm.name" 
              @focus="cardNameFocused = true"
              @blur="cardNameFocused = false; validateCardName()"
              placeholder="John Doe"
              class="form-input"
              :class="{ focused: cardNameFocused, error: cardNameError }"
              :disabled="isProcessing"
            />
            <div v-if="cardNameError" class="field-error">{{ cardNameError }}</div>
          </div>

          <div class="form-row">
            <div class="form-field">
              <label class="field-label">Expiry date</label>
              <input 
                type="text" 
                v-model="cardForm.expiry" 
                @input="formatExpiry"
                @focus="expiryFocused = true"
                @blur="expiryFocused = false; validateExpiry()"
                placeholder="MM/YY"
                maxlength="5"
                class="form-input"
                :class="{ focused: expiryFocused, error: expiryError }"
                :disabled="isProcessing"
              />
              <div v-if="expiryError" class="field-error">{{ expiryError }}</div>
            </div>
            <div class="form-field">
              <label class="field-label">
                CVV
                <span class="cvv-info" @mouseenter="showCvvTooltip = true" @mouseleave="showCvvTooltip = false">
                  <svg width="14" height="14" viewBox="0 0 14 14" fill="none">
                    <circle cx="7" cy="7" r="6" stroke="currentColor" stroke-width="1"/>
                    <path d="M7 5v3M7 9h.01" stroke="currentColor" stroke-width="1.5" stroke-linecap="round"/>
                  </svg>
                </span>
                <div v-if="showCvvTooltip" class="tooltip">3-digit code on the back of your card</div>
              </label>
              <input 
                type="text" 
                v-model="cardForm.cvv" 
                @input="formatCVV"
                @focus="cvvFocused = true"
                @blur="cvvFocused = false; validateCVV()"
                placeholder="123"
                maxlength="3"
                class="form-input"
                :class="{ focused: cvvFocused, error: cvvError }"
                :disabled="isProcessing"
              />
              <div v-if="cvvError" class="field-error">{{ cvvError }}</div>
            </div>
          </div>

          <!-- 保存卡信息选项 -->
          <div class="save-card-section">
            <label class="save-card-checkbox">
              <input 
                type="checkbox" 
                v-model="saveCard"
                :disabled="isProcessing"
              />
              <span class="save-card-text">
                <svg width="16" height="16" viewBox="0 0 16 16" fill="none" class="save-card-icon">
                  <path d="M8 1L3 3v4c0 3.5 2.5 6.5 5 7.5 2.5-1 5-4 5-7.5V3L8 1z" fill="currentColor"/>
                </svg>
                Save card for future payments
              </span>
            </label>
            <p class="save-card-desc">Your card details will be securely stored for faster checkout next time</p>
          </div>
        </div>

        <!-- 其他支付方式提示 -->
        <div v-if="paymentMethod && paymentMethod !== 'card'" class="payment-redirect-info">
          <div class="redirect-icon">{{ getPaymentMethodIcon(paymentMethod) }}</div>
          <p class="redirect-text">You will be redirected to {{ getPaymentMethodName(paymentMethod) }} to complete your payment.</p>
        </div>
      </div>

      <!-- 服务协议和隐私协议 -->
      <div class="terms-section">
        <label class="terms-checkbox">
          <input 
            type="checkbox" 
            v-model="agreedToTerms"
            :disabled="isProcessing"
          />
          <span class="terms-text">
            I agree to the 
            <a href="#" class="terms-link" @click.prevent="openTerms">Terms of Service</a>
            and 
            <a href="#" class="terms-link" @click.prevent="openPrivacy">Privacy Policy</a>
          </span>
        </label>
      </div>

      <!-- 安全标识 -->
      <div class="security-badge">
        <svg width="16" height="16" viewBox="0 0 16 16" fill="none">
          <path d="M8 1L3 3v4c0 3.5 2.5 6.5 5 7.5 2.5-1 5-4 5-7.5V3L8 1z" fill="currentColor"/>
        </svg>
        <span>Your payment information is secure and encrypted</span>
      </div>

      <!-- 底部操作栏 -->
      <div class="checkout-footer">
        <button 
          class="pay-button" 
          @click="handlePay"
          :disabled="isProcessing || !canSubmit"
          :class="{ processing: isProcessing }"
        >
          <span v-if="!isProcessing">Pay {{ formatCurrency(productPrice, currentCurrency) }}</span>
          <span v-else class="processing-spinner">
            <svg width="16" height="16" viewBox="0 0 16 16">
              <circle cx="8" cy="8" r="7" stroke="currentColor" stroke-width="2" fill="none" stroke-dasharray="44" stroke-dashoffset="11" class="spinner"/>
            </svg>
            Processing...
          </span>
        </button>
        <button class="cancel-link" @click="handleClose" :disabled="isProcessing">
          Cancel
        </button>
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
    type: Number,
    default: 9.99
  },
  defaultCurrency: {
    type: String,
    default: 'USD'
  },
  supportedCurrencies: {
    type: Array,
    default: () => ['USD', 'CNY', 'EUR', 'GBP', 'JPY']
  },
  supportedPaymentMethods: {
    type: Array,
    default: () => ['card', 'paypal', 'alipay', 'wechat']
  }
})

const emit = defineEmits(['close', 'payment-success', 'payment-error', 'jump-to-paypal', 'jump-to-card', 'currency-changed'])

const currentCurrency = ref(props.defaultCurrency)
const paymentMethod = ref('')
const isProcessing = ref(false)
const agreedToTerms = ref(true) // 默认同意
const saveCard = ref(false) // 保存卡信息选项

// 表单焦点状态
const cardNumberFocused = ref(false)
const cardNameFocused = ref(false)
const expiryFocused = ref(false)
const cvvFocused = ref(false)
const showCvvTooltip = ref(false)

// 表单错误
const cardNumberError = ref('')
const cardNameError = ref('')
const expiryError = ref('')
const cvvError = ref('')

const cardForm = ref({
  name: '',
  number: '',
  expiry: '',
  cvv: ''
})

// 支付方式配置
const paymentMethodConfig = {
  card: { 
    icon: '💳', 
    name: 'Credit or debit card', 
    description: 'Visa, Mastercard, Amex',
    enabled: true 
  },
  paypal: { 
    icon: '🔵', 
    name: 'PayPal', 
    description: 'Pay with your PayPal account',
    enabled: true,
    badge: 'Popular'
  },
  alipay: { 
    icon: '💙', 
    name: 'Alipay', 
    description: 'Pay with Alipay',
    enabled: true 
  },
  wechat: { 
    icon: '💚', 
    name: 'WeChat Pay', 
    description: 'Pay with WeChat',
    enabled: true 
  },
  applepay: { 
    icon: '🍎', 
    name: 'Apple Pay', 
    description: 'Pay with Apple Pay',
    enabled: false 
  },
  googlepay: { 
    icon: '📱', 
    name: 'Google Pay', 
    description: 'Pay with Google Pay',
    enabled: false 
  }
}

const availablePaymentMethods = computed(() => {
  return props.supportedPaymentMethods.map(id => ({
    id,
    ...paymentMethodConfig[id] || { icon: '🌐', name: id, enabled: false }
  }))
})

// 检测卡片品牌
const detectedCardBrand = computed(() => {
  const number = cardForm.value.number.replace(/\s/g, '')
  if (number.startsWith('4')) return 'Visa'
  if (number.startsWith('5') || number.startsWith('2')) return 'Mastercard'
  if (number.startsWith('3')) return 'Amex'
  if (number.startsWith('6')) return 'Discover'
  return ''
})

// 币种配置
const currencyConfig = {
  USD: { symbol: '$', rate: 1.0 },
  CNY: { symbol: '¥', rate: 7.2 },
  EUR: { symbol: '€', rate: 0.92 },
  GBP: { symbol: '£', rate: 0.79 },
  JPY: { symbol: '¥', rate: 150.0 }
}

const formatCurrency = (amount, currency) => {
  const config = currencyConfig[currency]
  if (!config) return `${amount} ${currency}`
  const convertedAmount = amount * (config.rate / currencyConfig[currentCurrency.value]?.rate || 1)
  return `${config.symbol}${convertedAmount.toFixed(2)}`
}

const selectPaymentMethod = (method) => {
  const methodInfo = paymentMethodConfig[method]
  if (methodInfo && methodInfo.enabled && !isProcessing.value) {
    paymentMethod.value = method
    // 清除错误
    clearErrors()
  }
}

const formatCardNumber = (event) => {
  let value = event.target.value.replace(/\s/g, '')
  value = value.replace(/\D/g, '')
  if (value.length > 16) {
    value = value.slice(0, 16)
  }
  const formatted = value.match(/.{1,4}/g)?.join(' ') || value
  cardForm.value.number = formatted
  cardNumberError.value = ''
}

const formatExpiry = (event) => {
  let value = event.target.value.replace(/\D/g, '')
  if (value.length > 4) {
    value = value.slice(0, 4)
  }
  if (value.length >= 2) {
    value = value.slice(0, 2) + '/' + value.slice(2)
  }
  cardForm.value.expiry = value
  expiryError.value = ''
}

const formatCVV = (event) => {
  let value = event.target.value.replace(/\D/g, '')
  if (value.length > 3) {
    value = value.slice(0, 3)
  }
  cardForm.value.cvv = value
  cvvError.value = ''
}

// 验证函数
const validateCardNumber = () => {
  const number = cardForm.value.number.replace(/\s/g, '')
  if (!number) {
    cardNumberError.value = 'Card number is required'
    return false
  }
  if (number.length < 13 || number.length > 19) {
    cardNumberError.value = 'Invalid card number'
    return false
  }
  // Luhn算法验证
  if (!luhnCheck(number)) {
    cardNumberError.value = 'Invalid card number'
    return false
  }
  cardNumberError.value = ''
  return true
}

const validateCardName = () => {
  if (!cardForm.value.name.trim()) {
    cardNameError.value = 'Cardholder name is required'
    return false
  }
  cardNameError.value = ''
  return true
}

const validateExpiry = () => {
  const expiry = cardForm.value.expiry.replace(/\D/g, '')
  if (!expiry) {
    expiryError.value = 'Expiry date is required'
    return false
  }
  if (expiry.length !== 4) {
    expiryError.value = 'Invalid expiry date'
    return false
  }
  const month = parseInt(expiry.slice(0, 2))
  const year = parseInt(expiry.slice(2, 4))
  const currentYear = new Date().getFullYear() % 100
  const currentMonth = new Date().getMonth() + 1
  
  if (month < 1 || month > 12) {
    expiryError.value = 'Invalid month'
    return false
  }
  if (year < currentYear || (year === currentYear && month < currentMonth)) {
    expiryError.value = 'Card has expired'
    return false
  }
  expiryError.value = ''
  return true
}

const validateCVV = () => {
  if (!cardForm.value.cvv) {
    cvvError.value = 'CVV is required'
    return false
  }
  if (cardForm.value.cvv.length < 3) {
    cvvError.value = 'Invalid CVV'
    return false
  }
  cvvError.value = ''
  return true
}

// Luhn算法验证
const luhnCheck = (number) => {
  let sum = 0
  let isEven = false
  for (let i = number.length - 1; i >= 0; i--) {
    let digit = parseInt(number[i])
    if (isEven) {
      digit *= 2
      if (digit > 9) digit -= 9
    }
    sum += digit
    isEven = !isEven
  }
  return sum % 10 === 0
}

const clearErrors = () => {
  cardNumberError.value = ''
  cardNameError.value = ''
  expiryError.value = ''
  cvvError.value = ''
}

const canSubmit = computed(() => {
  if (!paymentMethod.value) return false
  if (!agreedToTerms.value) return false // 必须同意协议
  if (paymentMethod.value === 'card') {
    return cardForm.value.name && 
           cardForm.value.number.replace(/\s/g, '').length >= 13 &&
           cardForm.value.expiry.replace(/\D/g, '').length === 4 &&
           cardForm.value.cvv.length >= 3 &&
           !cardNumberError.value &&
           !cardNameError.value &&
           !expiryError.value &&
           !cvvError.value
  }
  return true
})

const getPaymentMethodIcon = (method) => {
  return paymentMethodConfig[method]?.icon || '🌐'
}

const getPaymentMethodName = (method) => {
  return paymentMethodConfig[method]?.name || method
}

const handleOverlayClick = () => {
  if (!isProcessing.value) {
    handleClose()
  }
}

const handleClose = () => {
  if (!isProcessing.value) {
    emit('close')
  }
}

const handlePay = async () => {
  if (!canSubmit.value || isProcessing.value) return

  // 验证表单
  if (paymentMethod.value === 'card') {
    if (!validateCardNumber() || !validateCardName() || !validateExpiry() || !validateCVV()) {
      return
    }
  }

  isProcessing.value = true

  try {
    if (paymentMethod.value === 'paypal') {
      emit('jump-to-paypal')
      return
    }

    if (paymentMethod.value === 'card') {
      emit('jump-to-card', {
        name: cardForm.value.name,
        number: cardForm.value.number,
        expiry: cardForm.value.expiry,
        cvv: cardForm.value.cvv,
        currency: currentCurrency.value,
        saveCard: saveCard.value
      })
      return
    }

    await simulatePayment()
    emit('payment-success')
  } catch (error) {
    emit('payment-error', error.message)
  } finally {
    isProcessing.value = false
  }
}

const simulatePayment = () => {
  return new Promise((resolve) => {
    setTimeout(() => {
      resolve()
    }, 2000)
  })
}

const openTerms = () => {
  // 打开服务协议页面（实际应用中应该打开真实链接）
  window.open('#', '_blank')
}

const openPrivacy = () => {
  // 打开隐私政策页面（实际应用中应该打开真实链接）
  window.open('#', '_blank')
}

// 监听visible变化，重置状态
watch(() => props.visible, (newVal) => {
  if (newVal) {
    paymentMethod.value = ''
    cardForm.value = {
      name: '',
      number: '',
      expiry: '',
      cvv: ''
    }
    currentCurrency.value = props.defaultCurrency
    isProcessing.value = false
    agreedToTerms.value = true // 重置为默认同意
    saveCard.value = false // 重置保存卡信息选项
    clearErrors()
  }
})
</script>

<style scoped>
/* Adyen/Stripe风格 - 简洁专业 */
.checkout-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.6);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  padding: 20px;
  animation: fadeIn 0.2s ease;
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

.checkout-container {
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

@keyframes slideUp {
  from {
    transform: translateY(20px);
    opacity: 0;
  }
  to {
    transform: translateY(0);
    opacity: 1;
  }
}

/* 头部 */
.checkout-header {
  padding: 24px 24px 20px;
  border-bottom: 1px solid #e5e7eb;
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
}

.header-content {
  flex: 1;
}

.checkout-title {
  font-size: 20px;
  font-weight: 600;
  color: #111827;
  margin: 0 0 4px 0;
  line-height: 1.4;
}

.checkout-subtitle {
  font-size: 14px;
  color: #6b7280;
  margin: 0;
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
.order-summary {
  padding: 20px 24px;
  background-color: #f9fafb;
  border-bottom: 1px solid #e5e7eb;
}

.summary-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}

.summary-row:last-child {
  margin-bottom: 0;
}

.summary-label {
  font-size: 14px;
  color: #6b7280;
}

.summary-value {
  font-size: 14px;
  color: #111827;
  font-weight: 500;
}

.summary-amount {
  font-size: 18px;
  font-weight: 600;
}

/* 支付部分 */
.payment-section {
  padding: 24px;
  flex: 1;
  overflow-y: auto;
}

.section-title {
  font-size: 16px;
  font-weight: 600;
  color: #111827;
  margin: 0 0 16px 0;
}

/* 支付方式列表 */
.payment-methods-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-bottom: 24px;
}

.payment-method-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 16px;
  border: 1.5px solid #e5e7eb;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.15s;
  background: #ffffff;
}

.payment-method-item:hover:not(.disabled) {
  border-color: #d1d5db;
  background-color: #f9fafb;
}

.payment-method-item.active {
  border-color: #111827;
  background-color: #f9fafb;
}

.payment-method-item.disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.method-radio {
  flex-shrink: 0;
}

.radio-circle {
  width: 20px;
  height: 20px;
  border: 2px solid #d1d5db;
  border-radius: 50%;
  transition: all 0.15s;
  position: relative;
}

.radio-circle.checked {
  border-color: #111827;
}

.radio-circle.checked::after {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: 10px;
  height: 10px;
  background: #111827;
  border-radius: 50%;
}

.method-content {
  flex: 1;
  min-width: 0;
}

.method-header {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 4px;
}

.method-name {
  font-size: 15px;
  font-weight: 500;
  color: #111827;
}

.method-badge {
  font-size: 11px;
  font-weight: 600;
  color: #111827;
  background-color: #fef3c7;
  padding: 2px 6px;
  border-radius: 4px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.method-description {
  font-size: 13px;
  color: #6b7280;
}

.method-icon-wrapper {
  flex-shrink: 0;
}

.method-icon {
  font-size: 24px;
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #f9fafb;
  border-radius: 6px;
}

/* 卡片表单 */
.card-form-container {
  margin-top: 24px;
  padding-top: 24px;
  border-top: 1px solid #e5e7eb;
}

.form-field {
  margin-bottom: 20px;
}

.form-field:last-child {
  margin-bottom: 0;
}

.field-label {
  display: block;
  font-size: 13px;
  font-weight: 500;
  color: #374151;
  margin-bottom: 8px;
  position: relative;
}

.cvv-info {
  display: inline-flex;
  align-items: center;
  margin-left: 4px;
  color: #9ca3af;
  cursor: help;
  position: relative;
}

.tooltip {
  position: absolute;
  bottom: 100%;
  left: 50%;
  transform: translateX(-50%);
  margin-bottom: 8px;
  padding: 8px 12px;
  background: #111827;
  color: #fff;
  font-size: 12px;
  border-radius: 6px;
  white-space: nowrap;
  z-index: 10;
}

.tooltip::after {
  content: '';
  position: absolute;
  top: 100%;
  left: 50%;
  transform: translateX(-50%);
  border: 4px solid transparent;
  border-top-color: #111827;
}

.input-wrapper {
  position: relative;
  border: 1.5px solid #d1d5db;
  border-radius: 6px;
  background: #ffffff;
  transition: all 0.15s;
}

.input-wrapper.focused {
  border-color: #111827;
  box-shadow: 0 0 0 3px rgba(17, 24, 39, 0.1);
}

.input-wrapper.error {
  border-color: #ef4444;
}

.form-input,
.card-input {
  width: 100%;
  padding: 12px 16px;
  border: none;
  background: transparent;
  font-size: 15px;
  color: #111827;
  outline: none;
  font-family: inherit;
}

.form-input::placeholder,
.card-input::placeholder {
  color: #9ca3af;
}

.form-input:disabled,
.card-input:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.card-brands {
  position: absolute;
  right: 12px;
  top: 50%;
  transform: translateY(-50%);
  display: flex;
  align-items: center;
}

.card-brand {
  font-size: 12px;
  font-weight: 600;
  color: #6b7280;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.form-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 16px;
}

.field-error {
  margin-top: 6px;
  font-size: 13px;
  color: #ef4444;
  display: flex;
  align-items: center;
  gap: 4px;
}

/* 支付重定向信息 */
.payment-redirect-info {
  margin-top: 24px;
  padding: 20px;
  background-color: #f9fafb;
  border-radius: 8px;
  text-align: center;
}

.redirect-icon {
  font-size: 32px;
  margin-bottom: 12px;
}

.redirect-text {
  font-size: 14px;
  color: #6b7280;
  margin: 0;
  line-height: 1.5;
}

/* 服务协议和隐私协议 */
.terms-section {
  padding: 16px 24px;
  border-top: 1px solid #e5e7eb;
}

.terms-checkbox {
  display: flex;
  align-items: flex-start;
  gap: 10px;
  cursor: pointer;
  user-select: none;
}

.terms-checkbox input[type="checkbox"] {
  width: 18px;
  height: 18px;
  margin-top: 2px;
  cursor: pointer;
  accent-color: #111827;
  flex-shrink: 0;
}

.terms-checkbox input[type="checkbox"]:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.terms-text {
  font-size: 13px;
  color: #374151;
  line-height: 1.5;
}

.terms-link {
  color: #111827;
  text-decoration: underline;
  text-underline-offset: 2px;
  transition: color 0.15s;
}

.terms-link:hover {
  color: #374151;
}

/* 保存卡信息 */
.save-card-section {
  margin-top: 20px;
  padding-top: 20px;
  border-top: 1px solid #e5e7eb;
}

.save-card-checkbox {
  display: flex;
  align-items: center;
  gap: 10px;
  cursor: pointer;
  user-select: none;
  margin-bottom: 8px;
}

.save-card-checkbox input[type="checkbox"] {
  width: 18px;
  height: 18px;
  cursor: pointer;
  accent-color: #111827;
  flex-shrink: 0;
}

.save-card-checkbox input[type="checkbox"]:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.save-card-text {
  font-size: 14px;
  font-weight: 500;
  color: #111827;
  display: flex;
  align-items: center;
  gap: 6px;
}

.save-card-icon {
  color: #10b981;
  flex-shrink: 0;
}

.save-card-desc {
  font-size: 12px;
  color: #6b7280;
  margin: 0 0 0 28px;
  line-height: 1.4;
}

/* 安全标识 */
.security-badge {
  padding: 12px 24px;
  background-color: #f9fafb;
  border-top: 1px solid #e5e7eb;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  font-size: 12px;
  color: #6b7280;
}

.security-badge svg {
  color: #10b981;
  flex-shrink: 0;
}

/* 底部操作栏 */
.checkout-footer {
  padding: 20px 24px 24px;
  border-top: 1px solid #e5e7eb;
}

.pay-button {
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
  margin-bottom: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
}

.pay-button:hover:not(:disabled) {
  background-color: #1f2937;
}

.pay-button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.pay-button.processing {
  background-color: #6b7280;
}

.processing-spinner {
  display: flex;
  align-items: center;
  gap: 8px;
}

.spinner {
  animation: spin 0.8s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.cancel-link {
  width: 100%;
  padding: 10px;
  background: none;
  border: none;
  color: #6b7280;
  font-size: 14px;
  cursor: pointer;
  transition: color 0.15s;
}

.cancel-link:hover:not(:disabled) {
  color: #111827;
}

.cancel-link:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* 响应式设计 */
@media (max-width: 640px) {
  .checkout-container {
    max-width: 100%;
    border-radius: 0;
    max-height: 100vh;
  }

  .checkout-header {
    padding: 20px 20px 16px;
  }

  .payment-section {
    padding: 20px;
  }

  .form-row {
    grid-template-columns: 1fr;
  }
}
</style>

