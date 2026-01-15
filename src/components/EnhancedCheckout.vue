<template>
  <div v-if="visible" class="checkout-overlay" @click.self="handleOverlayClick">
    <div class="checkout-modal">
      <div class="checkout-header">
        <h2>{{ t('checkout.title') }}</h2>
        <div class="header-actions">
          <select v-model="currentLanguage" class="language-select" @change="changeLanguage">
            <option value="zh-CN">中文</option>
            <option value="en-US">English</option>
            <option value="ja-JP">日本語</option>
            <option value="ko-KR">한국어</option>
          </select>
          <button class="close-button" @click="handleClose">×</button>
        </div>
      </div>

      <div class="checkout-body">
        <!-- 商品信息 -->
        <div class="order-info">
          <div class="order-item">
            <span class="order-label">{{ t('checkout.productName') }}</span>
            <span class="order-value">{{ productName }}</span>
          </div>
          <div class="order-item">
            <span class="order-label">{{ t('checkout.orderAmount') }}</span>
            <span class="order-value order-price">
              {{ formatCurrency(productPrice, currentCurrency) }}
            </span>
          </div>
          <div class="order-item" v-if="exchangeRate">
            <span class="order-label">{{ t('checkout.exchangeRate') }}</span>
            <span class="order-value">{{ exchangeRate }}</span>
          </div>
        </div>

        <!-- 币种选择 -->
        <div class="currency-selector">
          <h3>{{ t('checkout.selectCurrency') }}</h3>
          <div class="currency-grid">
            <div 
              v-for="currency in availableCurrencies" 
              :key="currency.code"
              class="currency-option"
              :class="{ active: currentCurrency === currency.code }"
              @click="selectCurrency(currency.code)"
            >
              <span class="currency-flag">{{ currency.flag }}</span>
              <div class="currency-info">
                <div class="currency-code">{{ currency.code }}</div>
                <div class="currency-name">{{ currency.name }}</div>
              </div>
              <div class="currency-amount">
                {{ formatCurrency(productPrice, currency.code) }}
              </div>
            </div>
          </div>
        </div>

        <!-- 支付方式选择 -->
        <div class="payment-methods">
          <h3>{{ t('checkout.selectPaymentMethod') }}</h3>
          <div class="method-grid">
            <div 
              v-for="method in availablePaymentMethods" 
              :key="method.id"
              class="method-option" 
              :class="{ active: paymentMethod === method.id, disabled: !method.enabled }"
              @click="selectPaymentMethod(method.id)"
            >
              <div class="method-icon">{{ method.icon }}</div>
              <div class="method-name">{{ method.name }}</div>
              <div v-if="method.badge" class="method-badge">{{ method.badge }}</div>
            </div>
          </div>
        </div>

        <!-- 卡支付表单 -->
        <div v-if="paymentMethod === 'card' && cardPaymentEnabled" class="payment-form">
          <div class="form-group">
            <label>{{ t('checkout.cardholderName') }}</label>
            <input 
              type="text" 
              v-model="cardForm.name" 
              :placeholder="t('checkout.cardholderNamePlaceholder')"
              :disabled="isProcessing"
            />
          </div>
          <div class="form-group">
            <label>{{ t('checkout.cardNumber') }}</label>
            <input 
              type="text" 
              v-model="cardForm.number" 
              @input="formatCardNumber"
              placeholder="1234 5678 9012 3456"
              maxlength="19"
              :disabled="isProcessing"
            />
          </div>
          <div class="form-row">
            <div class="form-group">
              <label>{{ t('checkout.expiryDate') }}</label>
              <input 
                type="text" 
                v-model="cardForm.expiry" 
                @input="formatExpiry"
                placeholder="MM/YY"
                maxlength="5"
                :disabled="isProcessing"
              />
            </div>
            <div class="form-group">
              <label>{{ t('checkout.cvv') }}</label>
              <input 
                type="text" 
                v-model="cardForm.cvv" 
                @input="formatCVV"
                placeholder="123"
                maxlength="3"
                :disabled="isProcessing"
              />
            </div>
          </div>
        </div>

      </div>

      <div class="checkout-footer">
        <button class="cancel-button" @click="handleClose" :disabled="isProcessing">
          {{ t('checkout.cancel') }}
        </button>
        <button 
          class="confirm-button" 
          @click="handleConfirmPayment"
          :disabled="isProcessing || !canSubmit"
        >
          {{ isProcessing ? t('checkout.processing') : t('checkout.confirmPayment') }}
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
    default: '游戏道具'
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

const currentLanguage = ref('zh-CN')
const currentCurrency = ref(props.defaultCurrency)
const paymentMethod = ref('')
const isProcessing = ref(false)
const exchangeRate = ref(null)

// 多语言支持
const translations = {
  'zh-CN': {
    checkout: {
      title: '收银台',
      productName: '商品名称',
      orderAmount: '订单金额',
      exchangeRate: '汇率',
      selectCurrency: '选择币种',
      selectPaymentMethod: '选择支付方式',
      cardholderName: '持卡人姓名',
      cardholderNamePlaceholder: '请输入持卡人姓名',
      cardNumber: '卡号',
      expiryDate: '有效期',
      cvv: 'CVV',
      cancel: '取消',
      confirmPayment: '确认支付',
      processing: '处理中...'
    }
  },
  'en-US': {
    checkout: {
      title: 'Checkout',
      productName: 'Product Name',
      orderAmount: 'Order Amount',
      exchangeRate: 'Exchange Rate',
      selectCurrency: 'Select Currency',
      selectPaymentMethod: 'Select Payment Method',
      cardholderName: 'Cardholder Name',
      cardholderNamePlaceholder: 'Enter cardholder name',
      cardNumber: 'Card Number',
      expiryDate: 'Expiry Date',
      cvv: 'CVV',
      cancel: 'Cancel',
      confirmPayment: 'Confirm Payment',
      processing: 'Processing...'
    }
  },
  'ja-JP': {
    checkout: {
      title: 'レジ',
      productName: '商品名',
      orderAmount: '注文金額',
      exchangeRate: '為替レート',
      selectCurrency: '通貨を選択',
      selectPaymentMethod: '支払い方法を選択',
      cardholderName: 'カード名義人',
      cardholderNamePlaceholder: 'カード名義人を入力',
      cardNumber: 'カード番号',
      expiryDate: '有効期限',
      cvv: 'CVV',
      cancel: 'キャンセル',
      confirmPayment: '支払いを確認',
      processing: '処理中...'
    }
  },
  'ko-KR': {
    checkout: {
      title: '결제',
      productName: '상품명',
      orderAmount: '주문 금액',
      exchangeRate: '환율',
      selectCurrency: '통화 선택',
      selectPaymentMethod: '결제 수단 선택',
      cardholderName: '카드 소유자 이름',
      cardholderNamePlaceholder: '카드 소유자 이름을 입력하세요',
      cardNumber: '카드 번호',
      expiryDate: '유효 기간',
      cvv: 'CVV',
      cancel: '취소',
      confirmPayment: '결제 확인',
      processing: '처리 중...'
    }
  }
}

const t = (key) => {
  const keys = key.split('.')
  let value = translations[currentLanguage.value]
  for (const k of keys) {
    value = value?.[k]
  }
  return value || key
}

// 币种配置
const currencyConfig = {
  USD: { flag: '🇺🇸', name: '美元', symbol: '$', rate: 1.0 },
  CNY: { flag: '🇨🇳', name: '人民币', symbol: '¥', rate: 7.2 },
  EUR: { flag: '🇪🇺', name: '欧元', symbol: '€', rate: 0.92 },
  GBP: { flag: '🇬🇧', name: '英镑', symbol: '£', rate: 0.79 },
  JPY: { flag: '🇯🇵', name: '日元', symbol: '¥', rate: 150.0 }
}

const availableCurrencies = computed(() => {
  return props.supportedCurrencies.map(code => ({
    code,
    flag: currencyConfig[code]?.flag || '🌐',
    name: currencyConfig[code]?.name || code,
    symbol: currencyConfig[code]?.symbol || code
  }))
})

// 支付方式配置
const paymentMethodConfig = {
  card: { icon: '💳', name: '银行卡支付', enabled: true },
  paypal: { icon: '🔵', name: 'PayPal', enabled: true, badge: '推荐' },
  alipay: { icon: '💙', name: '支付宝', enabled: true },
  wechat: { icon: '💚', name: '微信支付', enabled: true },
  applepay: { icon: '🍎', name: 'Apple Pay', enabled: false },
  googlepay: { icon: '📱', name: 'Google Pay', enabled: false }
}

const availablePaymentMethods = computed(() => {
  return props.supportedPaymentMethods.map(id => ({
    id,
    ...paymentMethodConfig[id] || { icon: '🌐', name: id, enabled: false }
  }))
})

const cardPaymentEnabled = computed(() => {
  return props.supportedPaymentMethods.includes('card')
})

const cardForm = ref({
  name: '',
  number: '',
  expiry: '',
  cvv: ''
})

const canSubmit = computed(() => {
  if (!paymentMethod.value) return false
  if (paymentMethod.value === 'card') {
    return cardForm.value.name && 
           cardForm.value.number && 
           cardForm.value.expiry && 
           cardForm.value.cvv
  }
  return true
})

// 格式化货币
const formatCurrency = (amount, currency) => {
  const config = currencyConfig[currency]
  if (!config) return `${amount} ${currency}`
  
  const convertedAmount = amount * (config.rate / currencyConfig[currentCurrency.value]?.rate || 1)
  return `${config.symbol}${convertedAmount.toFixed(2)}`
}

// 选择币种
const selectCurrency = (currency) => {
  if (currency !== currentCurrency.value) {
    currentCurrency.value = currency
    emit('currency-changed', currency)
    // 这里可以调用API获取汇率
    exchangeRate.value = `1 ${props.defaultCurrency} = ${currencyConfig[currency]?.rate || 1} ${currency}`
  }
}

// 选择支付方式
const selectPaymentMethod = (method) => {
  const methodInfo = paymentMethodConfig[method]
  if (methodInfo && methodInfo.enabled && !isProcessing.value) {
    paymentMethod.value = method
  }
}

// 切换语言
const changeLanguage = () => {
  // 语言切换逻辑
}

// 格式化卡号
const formatCardNumber = (event) => {
  let value = event.target.value.replace(/\s/g, '')
  value = value.replace(/\D/g, '')
  if (value.length > 16) {
    value = value.slice(0, 16)
  }
  const formatted = value.match(/.{1,4}/g)?.join(' ') || value
  cardForm.value.number = formatted
}

// 格式化有效期
const formatExpiry = (event) => {
  let value = event.target.value.replace(/\D/g, '')
  if (value.length > 4) {
    value = value.slice(0, 4)
  }
  if (value.length >= 2) {
    value = value.slice(0, 2) + '/' + value.slice(2)
  }
  cardForm.value.expiry = value
}

// 格式化CVV
const formatCVV = (event) => {
  let value = event.target.value.replace(/\D/g, '')
  if (value.length > 3) {
    value = value.slice(0, 3)
  }
  cardForm.value.cvv = value
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

const handleConfirmPayment = async () => {
  if (!canSubmit.value || isProcessing.value) return

  isProcessing.value = true

  try {
    // 如果是PayPal支付，跳转到PayPal页面
    if (paymentMethod.value === 'paypal') {
      emit('jump-to-paypal')
      return
    }

    // 银行卡支付，跳转到卡支付页面
    if (paymentMethod.value === 'card') {
      emit('jump-to-card', {
        name: cardForm.value.name,
        number: cardForm.value.number,
        expiry: cardForm.value.expiry,
        cvv: cardForm.value.cvv,
        currency: currentCurrency.value
      })
      return
    }

    // 其他支付方式
    await simulatePayment()
    emit('payment-success')
  } catch (error) {
    emit('payment-error', error.message)
  } finally {
    isProcessing.value = false
  }
}

const simulatePayment = () => {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      resolve()
    }, 2000)
  })
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
  }
})
</script>

<style scoped>
.checkout-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  animation: fadeIn 0.3s ease;
}

@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

.checkout-modal {
  background-color: #ffffff;
  border-radius: 12px;
  width: 90%;
  max-width: 600px;
  max-height: 90vh;
  display: flex;
  flex-direction: column;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
  animation: slideUp 0.3s ease;
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

.checkout-header {
  padding: 20px 24px;
  border-bottom: 1px solid #e0e0e0;
  display: flex;
  justify-content: space-between;
  align-items: center;
  background-color: #fafafa;
  border-radius: 12px 12px 0 0;
}

.checkout-header h2 {
  font-size: 20px;
  font-weight: 600;
  color: #333;
  margin: 0;
}

.header-actions {
  display: flex;
  align-items: center;
  gap: 12px;
}

.language-select {
  padding: 6px 12px;
  border: 1px solid #e0e0e0;
  border-radius: 6px;
  font-size: 14px;
  background: #fff;
  cursor: pointer;
}

.close-button {
  background: none;
  border: none;
  font-size: 28px;
  color: #999;
  cursor: pointer;
  padding: 0;
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: color 0.2s;
}

.close-button:hover {
  color: #333;
}

.checkout-body {
  padding: 24px;
  overflow-y: auto;
  flex: 1;
}

.order-info {
  background-color: #f5f5f5;
  border-radius: 8px;
  padding: 16px;
  margin-bottom: 24px;
}

.order-item {
  display: flex;
  justify-content: space-between;
  margin-bottom: 8px;
}

.order-item:last-child {
  margin-bottom: 0;
}

.order-label {
  color: #666;
  font-size: 14px;
}

.order-value {
  color: #333;
  font-size: 14px;
  font-weight: 500;
}

.order-price {
  font-size: 18px;
  font-weight: 600;
  color: #333;
}

.currency-selector {
  margin-bottom: 24px;
}

.currency-selector h3,
.payment-methods h3 {
  font-size: 16px;
  font-weight: 600;
  color: #333;
  margin-bottom: 12px;
}

.currency-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(120px, 1fr));
  gap: 12px;
}

.currency-option {
  border: 2px solid #e0e0e0;
  border-radius: 8px;
  padding: 12px;
  text-align: center;
  cursor: pointer;
  transition: all 0.2s;
  background-color: #ffffff;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
}

.currency-option:hover {
  border-color: #999;
}

.currency-option.active {
  border-color: #1890ff;
  background-color: #f0f7ff;
}

.currency-flag {
  font-size: 24px;
}

.currency-code {
  font-size: 14px;
  font-weight: 600;
  color: #333;
}

.currency-name {
  font-size: 12px;
  color: #666;
}

.currency-amount {
  font-size: 13px;
  font-weight: 500;
  color: #1890ff;
}

.method-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(120px, 1fr));
  gap: 12px;
}

.method-option {
  border: 2px solid #e0e0e0;
  border-radius: 8px;
  padding: 16px;
  text-align: center;
  cursor: pointer;
  transition: all 0.2s;
  background-color: #ffffff;
  position: relative;
}

.method-option:hover:not(.disabled) {
  border-color: #999;
}

.method-option.active {
  border-color: #1890ff;
  background-color: #f0f7ff;
}

.method-option.disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.method-icon {
  font-size: 32px;
  margin-bottom: 8px;
}

.method-name {
  font-size: 14px;
  color: #333;
  font-weight: 500;
}

.method-badge {
  position: absolute;
  top: 8px;
  right: 8px;
  background-color: #ff4d4f;
  color: #fff;
  font-size: 10px;
  padding: 2px 6px;
  border-radius: 4px;
}

.payment-form {
  margin-top: 20px;
}

.form-group {
  margin-bottom: 16px;
}

.form-group label {
  display: block;
  font-size: 14px;
  color: #333;
  margin-bottom: 8px;
  font-weight: 500;
}

.form-group input {
  width: 100%;
  padding: 12px;
  border: 1px solid #e0e0e0;
  border-radius: 6px;
  font-size: 14px;
  color: #333;
  transition: border-color 0.2s;
}

.form-group input:focus {
  outline: none;
  border-color: #1890ff;
}

.form-group input:disabled {
  background-color: #f5f5f5;
  cursor: not-allowed;
}

.form-row {
  display: flex;
  gap: 12px;
}

.form-row .form-group {
  flex: 1;
}

.checkout-footer {
  padding: 20px 24px;
  border-top: 1px solid #e0e0e0;
  display: flex;
  gap: 12px;
  background-color: #fafafa;
  border-radius: 0 0 12px 12px;
}

.cancel-button,
.confirm-button {
  flex: 1;
  padding: 12px 24px;
  border: none;
  border-radius: 6px;
  font-size: 16px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
}

.cancel-button {
  background-color: #f5f5f5;
  color: #333;
}

.cancel-button:hover:not(:disabled) {
  background-color: #e0e0e0;
}

.confirm-button {
  background-color: #1890ff;
  color: #ffffff;
}

.confirm-button:hover:not(:disabled) {
  background-color: #40a9ff;
}

.confirm-button:disabled,
.cancel-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
</style>

