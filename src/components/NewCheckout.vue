<template>
  <div v-if="visible" class="checkout-overlay" @click.self="handleOverlayClick">
    <div class="checkout-container">
      <!-- 头部 -->
      <div class="checkout-header">
        <div class="header-content">
          <h1 class="checkout-title">Complete your payment</h1>
          <p class="checkout-subtitle">Order #{{ orderId || 'Loading...' }}</p>
        </div>
        <button class="close-btn" @click="handleClose" aria-label="Close" :disabled="isProcessing">
          <svg width="20" height="20" viewBox="0 0 20 20" fill="none">
            <path d="M15 5L5 15M5 5l10 10" stroke="currentColor" stroke-width="2" stroke-linecap="round"/>
          </svg>
        </button>
      </div>

      <!-- 加载状态 -->
      <div v-if="loadingOrder" class="loading-section">
        <div class="loading-spinner"></div>
        <p>Creating order...</p>
      </div>

      <!-- 订单信息 -->
      <div v-else-if="orderInfo" class="order-section">
        <!-- 订单摘要 -->
        <div class="order-summary">
          <div class="summary-row">
            <span class="summary-label">Item</span>
            <span class="summary-value">{{ orderInfo.goodsName || productName }}</span>
          </div>
          <div class="summary-row">
            <span class="summary-label">Amount</span>
            <span class="summary-value summary-amount">
              {{ formatCurrency(orderInfo.price || productPrice, orderInfo.currency || currentCurrency) }}
            </span>
          </div>
          <div v-if="orderInfo.discount && parseFloat(orderInfo.discount) < 1" class="summary-row discount">
            <span class="summary-label">Discount</span>
            <span class="summary-value discount-text">{{ orderInfo.discountText || `${(1 - parseFloat(orderInfo.discount)) * 100}% off` }}</span>
          </div>
        </div>

        <!-- Web支付模式 -->
        <div v-if="webPayInfo && webPayInfo.enableWebPay" class="web-pay-section">
          <h2 class="section-title">Payment method</h2>
          <div class="payment-channels">
            <div 
              v-for="channel in webPayInfo.payChannels" 
              :key="channel.channelName"
              class="channel-item"
              :class="{ active: selectedChannel === channel.channelName, recommend: channel.recommend }"
              @click="selectChannel(channel)"
            >
              <div class="channel-content">
                <div class="channel-header">
                  <span class="channel-name">{{ getChannelName(channel.channelName) }}</span>
                  <span v-if="channel.recommend" class="recommend-badge">Recommended</span>
                </div>
                <div class="channel-price">
                  <span class="original-price" v-if="channel.discount && parseFloat(channel.discount) < 1">
                    {{ formatCurrency(channel.price, channel.currency) }}
                  </span>
                  <span class="final-price">
                    {{ formatCurrency(channel.discountPrice || channel.price, channel.currency) }}
                  </span>
                </div>
                <div v-if="channel.discountText" class="channel-discount">{{ channel.discountText }}</div>
              </div>
              <div class="channel-icon">
                <img v-if="channel.icon" :src="channel.icon" :alt="channel.name" />
                <span v-else>{{ getChannelIcon(channel.channelName) }}</span>
              </div>
            </div>
          </div>
          <button 
            class="pay-button" 
            @click="handleWebPay"
            :disabled="!selectedChannel || isProcessing"
            :class="{ processing: isProcessing }"
          >
            <span v-if="!isProcessing">Pay {{ formatCurrency(orderInfo.price || productPrice, orderInfo.currency || currentCurrency) }}</span>
            <span v-else class="processing-spinner">
              <svg width="16" height="16" viewBox="0 0 16 16">
                <circle cx="8" cy="8" r="7" stroke="currentColor" stroke-width="2" fill="none" stroke-dasharray="44" stroke-dashoffset="11" class="spinner"/>
              </svg>
              Processing...
            </span>
          </button>
        </div>

        <!-- 原生支付模式 -->
        <div v-else class="native-pay-section">
          <h2 class="section-title">Payment method</h2>
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
                <label class="field-label">CVV</label>
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
        </div>

        <!-- Token信息 -->
        <div v-if="orderToken" class="token-info">
          <div class="token-expiry">
            <span class="token-label">Order expires in:</span>
            <span class="token-time">{{ formatExpiryTime(orderToken.expire) }}</span>
          </div>
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
          v-if="!webPayInfo || !webPayInfo.enableWebPay"
          class="pay-button" 
          @click="handlePay"
          :disabled="isProcessing || !canSubmit"
          :class="{ processing: isProcessing }"
        >
          <span v-if="!isProcessing">Pay {{ formatCurrency(orderInfo?.price || productPrice, orderInfo?.currency || currentCurrency) }}</span>
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
import { ref, computed, watch, onMounted, onUnmounted } from 'vue'

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
  goodsId: {
    type: String,
    default: ''
  },
  userId: {
    type: String,
    required: true
  },
  appId: {
    type: String,
    required: true
  },
  platform: {
    type: String,
    default: 'pc'
  },
  gameEnv: {
    type: String,
    default: 'pc'
  },
  currency: {
    type: String,
    default: 'USD'
  },
  language: {
    type: String,
    default: 'en'
  },
  location: {
    type: String,
    default: 'US'
  }
})

const emit = defineEmits(['close', 'payment-success', 'payment-error', 'order-created', 'jump-to-paypal', 'jump-to-card', 'jump-to-web'])

const loadingOrder = ref(false)
const orderId = ref(null)
const orderInfo = ref(null)
const orderToken = ref(null)
const webPayInfo = ref(null)
const paymentMethod = ref('')
const selectedChannel = ref('')
const isProcessing = ref(false)
const agreedToTerms = ref(true) // 默认同意
const saveCard = ref(false) // 保存卡信息选项

// 表单状态
const cardNumberFocused = ref(false)
const cardNameFocused = ref(false)
const expiryFocused = ref(false)
const cvvFocused = ref(false)

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

// Token刷新定时器
let tokenRefreshTimer = null
let expiryCheckTimer = null

const currentCurrency = computed(() => orderInfo.value?.currency || props.currency)

// 支付方式配置
const paymentMethodConfig = {
  card: { icon: '💳', name: 'Credit or debit card', description: 'Visa, Mastercard, Amex', enabled: true },
  paypal: { icon: '🔵', name: 'PayPal', description: 'Pay with your PayPal account', enabled: true, badge: 'Popular' },
  alipay: { icon: '💙', name: 'Alipay', description: 'Pay with Alipay', enabled: true },
  wechat: { icon: '💚', name: 'WeChat Pay', description: 'Pay with WeChat', enabled: true }
}

const availablePaymentMethods = computed(() => {
  // 根据平台和地区返回可用支付方式
  const methods = ['card', 'paypal']
  if (props.location === 'CN') {
    methods.push('alipay', 'wechat')
  }
  return methods.map(id => ({
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

const formatCurrency = (amount, currency) => {
  const symbols = {
    USD: '$', CNY: '¥', EUR: '€', GBP: '£', JPY: '¥'
  }
  return `${symbols[currency] || currency}${parseFloat(amount).toFixed(2)}`
}

const formatExpiryTime = (expire) => {
  if (!expire) return ''
  const now = Date.now()
  const expireTime = parseInt(expire)
  const diff = expireTime - now
  if (diff <= 0) return 'Expired'
  const minutes = Math.floor(diff / 60000)
  const seconds = Math.floor((diff % 60000) / 1000)
  return `${minutes}:${seconds.toString().padStart(2, '0')}`
}

const getChannelName = (channelName) => {
  const names = {
    wechat_jsapi: 'WeChat Pay',
    alipay_h5: 'Alipay',
    alipay_global_pc: 'Alipay Global',
    wechat: 'WeChat Pay',
    alipay: 'Alipay'
  }
  return names[channelName] || channelName
}

const getChannelIcon = (channelName) => {
  if (channelName.includes('wechat')) return '💚'
  if (channelName.includes('alipay')) return '💙'
  return '💳'
}

// 创建订单（safeInit3）
const createOrder = async () => {
  loadingOrder.value = true
  try {
    // 调用safeInit3接口
    const response = await fetch('/apiv3/payment/safeInit3', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded'
      },
      body: new URLSearchParams({
        channelName: 'he',
        data: encryptRequestData({
          platform: props.platform,
          channelName: 'he',
          appId: props.appId,
          userId: props.userId,
          goodsId: props.goodsId || 'default',
          goodsName: props.productName,
          price: props.productPrice.toString(),
          currency: props.currency,
          language: props.language,
          location: props.location,
          gameEnv: props.gameEnv,
          orderSource: props.gameEnv,
          delay_prepay: true
        })
      })
    })

    const result = await response.json()
    if (result.code === 0) {
      const data = decryptResponseData(result.ret)
      orderId.value = data.orderId
      orderInfo.value = {
        goodsName: props.productName,
        price: props.productPrice,
        currency: props.currency
      }
      orderToken.value = data.orderToken
      webPayInfo.value = data.webPayInfo

      emit('order-created', {
        orderId: data.orderId,
        orderToken: data.orderToken,
        webPayInfo: data.webPayInfo
      })

      // 启动token刷新
      if (data.orderToken) {
        startTokenRefresh(data.orderToken)
      }
    } else {
      throw new Error(result.message || 'Failed to create order')
    }
  } catch (error) {
    emit('payment-error', error.message)
  } finally {
    loadingOrder.value = false
  }
}

// Token刷新
const refreshToken = async () => {
  if (!orderId.value) return
  
  try {
    const response = await fetch('/apiv3/service/order/token', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded'
      },
      body: new URLSearchParams({
        orderId: orderId.value.toString()
      })
    })

    const result = await response.json()
    if (result.code === 0 && result.ret) {
      orderToken.value = {
        token: result.ret.token,
        expire: result.ret.expire,
        duration: result.ret.duration
      }
    }
  } catch (error) {
    console.error('Token refresh failed:', error)
  }
}

const startTokenRefresh = (token) => {
  // 清除旧定时器
  if (tokenRefreshTimer) {
    clearInterval(tokenRefreshTimer)
  }
  if (expiryCheckTimer) {
    clearInterval(expiryCheckTimer)
  }

  // 检查过期时间
  expiryCheckTimer = setInterval(() => {
    if (orderToken.value) {
      const expire = parseInt(orderToken.value.expire)
      const now = Date.now()
      const diff = expire - now
      
      // 如果剩余时间少于30秒，刷新token
      if (diff < 30000 && diff > 0) {
        refreshToken()
      }
      
      // 如果已过期，提示用户
      if (diff <= 0) {
        emit('payment-error', 'Order expired. Please create a new order.')
      }
    }
  }, 1000)

  // 定期刷新token（在过期前1分钟）
  const duration = parseInt(token.duration) * 1000
  const refreshTime = duration - 60000 // 提前1分钟刷新
  
  if (refreshTime > 0) {
    tokenRefreshTimer = setTimeout(() => {
      refreshToken()
    }, refreshTime)
  }
}

const selectChannel = (channel) => {
  if (!isProcessing.value) {
    selectedChannel.value = channel.channelName
  }
}

const selectPaymentMethod = (method) => {
  const methodInfo = paymentMethodConfig[method]
  if (methodInfo && methodInfo.enabled && !isProcessing.value) {
    paymentMethod.value = method
    clearErrors()
  }
}

const formatCardNumber = (event) => {
  let value = event.target.value.replace(/\s/g, '')
  value = value.replace(/\D/g, '')
  // 根据卡类型限制长度：Amex 15位，其他最多19位
  const maxLength = detectedCardBrand.value === 'Amex' ? 15 : 19
  if (value.length > maxLength) {
    value = value.slice(0, maxLength)
  }
  // Amex卡号格式：4-6-5，其他：4-4-4-4
  let formatted = value
  if (detectedCardBrand.value === 'Amex') {
    if (value.length > 4) {
      formatted = value.slice(0, 4) + ' ' + value.slice(4)
    }
    if (value.length > 10) {
      formatted = value.slice(0, 4) + ' ' + value.slice(4, 10) + ' ' + value.slice(10)
    }
  } else {
    formatted = value.match(/.{1,4}/g)?.join(' ') || value
  }
  cardForm.value.number = formatted
  // 清除错误信息（如果正在输入）
  if (cardNumberFocused.value) {
  cardNumberError.value = ''
  }
}

const formatExpiry = (event) => {
  let value = event.target.value.replace(/\D/g, '')
  if (value.length > 4) value = value.slice(0, 4)
  if (value.length >= 2) {
    value = value.slice(0, 2) + '/' + value.slice(2)
  }
  cardForm.value.expiry = value
  // 清除错误信息（如果正在输入）
  if (expiryFocused.value) {
  expiryError.value = ''
  }
}

const formatCVV = (event) => {
  let value = event.target.value.replace(/\D/g, '')
  // Amex卡CVV是4位，其他是3位
  const maxLength = detectedCardBrand.value === 'Amex' ? 4 : 3
  if (value.length > maxLength) {
    value = value.slice(0, maxLength)
  }
  cardForm.value.cvv = value
  // 清除错误信息（如果正在输入）
  if (cvvFocused.value) {
  cvvError.value = ''
  }
}

const validateCardNumber = () => {
  const number = cardForm.value.number.replace(/\s/g, '')
  if (!number) {
    cardNumberError.value = 'Card number is required'
    return false
  }
  if (number.length < 13) {
    cardNumberError.value = 'Card number is too short. Please enter the complete card number'
    return false
  }
  // Amex卡15位，其他13-19位
  if (detectedCardBrand.value === 'Amex') {
    if (number.length !== 15) {
      cardNumberError.value = 'American Express card number must be 15 digits'
      return false
    }
  } else {
  if (number.length < 13 || number.length > 19) {
      cardNumberError.value = 'Card number must be between 13 and 19 digits'
    return false
  }
  }
  // Luhn算法验证
  if (!luhnCheck(number)) {
    cardNumberError.value = 'Invalid card number. Please check and try again'
    return false
  }
  cardNumberError.value = ''
  return true
}

const validateCardName = () => {
  const name = cardForm.value.name.trim()
  if (!name) {
    cardNameError.value = 'Cardholder name is required'
    return false
  }
  if (name.length < 2) {
    cardNameError.value = 'Cardholder name must be at least 2 characters'
    return false
  }
  if (name.length > 50) {
    cardNameError.value = 'Cardholder name cannot exceed 50 characters'
    return false
  }
  // 检查是否包含数字
  if (/\d/.test(name)) {
    cardNameError.value = 'Cardholder name cannot contain numbers'
    return false
  }
  // 检查是否包含特殊字符（允许空格、连字符、撇号）
  if (!/^[a-zA-Z\s\-'\.]+$/.test(name)) {
    cardNameError.value = 'Cardholder name can only contain letters, spaces, hyphens, and apostrophes'
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
    expiryError.value = 'Invalid expiry date format. Please enter MM/YY'
    return false
  }
  const month = parseInt(expiry.slice(0, 2))
  const year = parseInt(expiry.slice(2, 4))
  const currentYear = new Date().getFullYear() % 100
  const currentMonth = new Date().getMonth() + 1
  
  if (month < 1 || month > 12) {
    expiryError.value = 'Invalid month. Please enter a month between 01 and 12'
    return false
  }
  if (year < currentYear || (year === currentYear && month < currentMonth)) {
    expiryError.value = 'This card has expired. Please use a valid expiry date'
    return false
  }
  expiryError.value = ''
  return true
}

const validateCVV = () => {
  const cvv = cardForm.value.cvv
  if (!cvv) {
    cvvError.value = 'CVV is required'
    return false
  }
  // Amex卡CVV是4位，其他是3位
  const requiredLength = detectedCardBrand.value === 'Amex' ? 4 : 3
  if (cvv.length !== requiredLength) {
    cvvError.value = detectedCardBrand.value === 'Amex' 
      ? 'CVV must be 4 digits for American Express cards' 
      : 'CVV must be 3 digits'
    return false
  }
  if (!/^\d+$/.test(cvv)) {
    cvvError.value = 'CVV can only contain numbers'
    return false
  }
  cvvError.value = ''
  return true
}

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
  if (!agreedToTerms.value) return false // 必须同意协议
  if (webPayInfo.value && webPayInfo.value.enableWebPay) {
    return !!selectedChannel.value
  }
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
  return !!paymentMethod.value
})

const handleWebPay = async () => {
  if (!selectedChannel.value || isProcessing.value) return
  
  isProcessing.value = true
  try {
    const channel = webPayInfo.value.payChannels.find(c => c.channelName === selectedChannel.value)
    if (channel && channel.webUrl) {
      // 构建支付URL
      const payUrl = buildPayUrl(channel)
      emit('jump-to-web', {
        url: payUrl,
        channelName: selectedChannel.value,
        orderId: orderId.value
      })
    } else {
      throw new Error('Payment channel not available')
    }
  } catch (error) {
    emit('payment-error', error.message)
  } finally {
    isProcessing.value = false
  }
}

const buildPayUrl = (channel) => {
  const baseUrl = channel.webUrl
  const params = new URLSearchParams({
    orderId: orderId.value.toString(),
    appId: props.appId,
    platform: props.platform,
    dcAppId: props.appId,
    goodsName: props.productName,
    price: (orderInfo.value?.price || props.productPrice).toString(),
    gameName: props.appId,
    currency: orderInfo.value?.currency || props.currency,
    channelName: channel.channelName,
    devicePlatform: props.gameEnv,
    userId: props.userId,
    token: orderToken.value?.token || '',
    source: props.gameEnv
  })
  return `${baseUrl}?${params.toString()}`
}

const handlePay = async () => {
  if (!canSubmit.value || isProcessing.value) return

  if (paymentMethod.value === 'card') {
    if (!validateCardNumber() || !validateCardName() || !validateExpiry() || !validateCVV()) {
      return
    }
  }

  isProcessing.value = true

  try {
    if (paymentMethod.value === 'paypal') {
      emit('jump-to-paypal', { orderId: orderId.value })
      return
    }

    if (paymentMethod.value === 'card') {
      // 调用safePrepay接口
      await callPrepay()
      emit('jump-to-card', {
        orderId: orderId.value,
        name: cardForm.value.name,
        number: cardForm.value.number,
        expiry: cardForm.value.expiry,
        cvv: cardForm.value.cvv,
        currency: currentCurrency.value,
        saveCard: saveCard.value
      })
      return
    }

    emit('payment-success', { orderId: orderId.value })
  } catch (error) {
    emit('payment-error', error.message)
  } finally {
    isProcessing.value = false
  }
}

const callPrepay = async () => {
  if (!orderId.value || !orderToken.value) {
    throw new Error('Order not initialized')
  }

  const response = await fetch('/apiv3/payment/safePrepay', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/x-www-form-urlencoded'
    },
    body: new URLSearchParams({
      channelName: 'he',
      data: encryptRequestData({
        appId: props.appId,
        platform: props.platform,
        channelName: 'he',
        userId: props.userId,
        orderId: orderId.value,
        token: orderToken.value.token,
        orderSource: props.gameEnv
      })
    })
  })

  const result = await response.json()
  if (result.code !== 0) {
    throw new Error(result.message || 'Prepay failed')
  }
  
  return decryptResponseData(result.ret)
}

const handleOverlayClick = () => {
  if (!isProcessing.value) {
    handleClose()
  }
}

const handleClose = () => {
  if (!isProcessing.value) {
    // 清理定时器
    if (tokenRefreshTimer) {
      clearTimeout(tokenRefreshTimer)
      tokenRefreshTimer = null
    }
    if (expiryCheckTimer) {
      clearInterval(expiryCheckTimer)
      expiryCheckTimer = null
    }
    emit('close')
  }
}

// 加密/解密函数（需要根据实际实现）
const encryptRequestData = (data) => {
  // TODO: 实现AES 192加密
  return JSON.stringify(data)
}

const decryptResponseData = (data) => {
  // TODO: 实现AES 192解密
  try {
    return JSON.parse(data)
  } catch {
    return {}
  }
}

const openTerms = () => {
  // 打开服务协议页面（实际应用中应该打开真实链接）
  window.open('#', '_blank')
}

const openPrivacy = () => {
  // 打开隐私政策页面（实际应用中应该打开真实链接）
  window.open('#', '_blank')
}

// 监听visible变化
watch(() => props.visible, (newVal) => {
  if (newVal) {
    // 重置状态
    orderId.value = null
    orderInfo.value = null
    orderToken.value = null
    webPayInfo.value = null
    paymentMethod.value = ''
    selectedChannel.value = ''
    isProcessing.value = false
    agreedToTerms.value = true // 重置为默认同意
    saveCard.value = false // 重置保存卡信息选项
    clearErrors()
    
    // 创建订单
    createOrder()
  } else {
    // 清理定时器
    if (tokenRefreshTimer) {
      clearTimeout(tokenRefreshTimer)
      tokenRefreshTimer = null
    }
    if (expiryCheckTimer) {
      clearInterval(expiryCheckTimer)
      expiryCheckTimer = null
    }
  }
})

onUnmounted(() => {
  if (tokenRefreshTimer) {
    clearTimeout(tokenRefreshTimer)
  }
  if (expiryCheckTimer) {
    clearInterval(expiryCheckTimer)
  }
})
</script>

<style scoped>
/* 复用Adyen/Stripe风格样式，并添加新样式 */
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

.close-btn:hover:not(:disabled) {
  color: #111827;
}

.close-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.loading-section {
  padding: 60px 24px;
  text-align: center;
}

.loading-spinner {
  width: 48px;
  height: 48px;
  border: 4px solid #e5e7eb;
  border-top-color: #111827;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
  margin: 0 auto 16px;
}

.order-section {
  flex: 1;
  overflow-y: auto;
}

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

.summary-row.discount {
  color: #10b981;
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

.discount-text {
  color: #10b981;
  font-weight: 600;
}

.web-pay-section,
.native-pay-section {
  padding: 24px;
}

.section-title {
  font-size: 16px;
  font-weight: 600;
  color: #111827;
  margin: 0 0 16px 0;
}

.payment-channels {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-bottom: 24px;
}

.channel-item {
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 16px;
  border: 1.5px solid #e5e7eb;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.15s;
  background: #ffffff;
}

.channel-item:hover {
  border-color: #d1d5db;
  background-color: #f9fafb;
}

.channel-item.active {
  border-color: #111827;
  background-color: #f9fafb;
}

.channel-item.recommend {
  border-color: #fef3c7;
  background-color: #fffbeb;
}

.channel-content {
  flex: 1;
}

.channel-header {
  display: flex;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
}

.channel-name {
  font-size: 15px;
  font-weight: 500;
  color: #111827;
}

.recommend-badge {
  font-size: 11px;
  font-weight: 600;
  color: #111827;
  background-color: #fef3c7;
  padding: 2px 6px;
  border-radius: 4px;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.channel-price {
  display: flex;
  align-items: baseline;
  gap: 8px;
  margin-bottom: 4px;
}

.original-price {
  font-size: 14px;
  color: #9ca3af;
  text-decoration: line-through;
}

.final-price {
  font-size: 16px;
  font-weight: 600;
  color: #111827;
}

.channel-discount {
  font-size: 12px;
  color: #10b981;
}

.channel-icon {
  width: 48px;
  height: 48px;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #f9fafb;
  border-radius: 6px;
  flex-shrink: 0;
}

.channel-icon img {
  width: 100%;
  height: 100%;
  object-fit: contain;
}

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

.form-input.focused {
  border-color: #111827;
}

.form-input.error {
  border-color: #ef4444;
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
  animation: slideDown 0.2s ease;
}

.field-error::before {
  content: '⚠';
  font-size: 14px;
}

@keyframes slideDown {
  from {
    opacity: 0;
    transform: translateY(-4px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.token-info {
  padding: 12px 24px;
  background-color: #f9fafb;
  border-top: 1px solid #e5e7eb;
}

.token-expiry {
  display: flex;
  justify-content: space-between;
  align-items: center;
  font-size: 12px;
}

.token-label {
  color: #6b7280;
}

.token-time {
  color: #111827;
  font-weight: 500;
  font-family: monospace;
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

@media (max-width: 640px) {
  .checkout-container {
    max-width: 100%;
    border-radius: 0;
    max-height: 100vh;
  }

  .checkout-header {
    padding: 20px 20px 16px;
  }

  .web-pay-section,
  .native-pay-section {
    padding: 20px;
  }

  .form-row {
    grid-template-columns: 1fr;
  }
}
</style>

