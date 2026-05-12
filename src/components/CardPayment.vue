<template>
  <div v-if="visible" class="card-payment-overlay">
    <div class="card-payment-container">
      <!-- 头部 -->
      <div class="payment-header">
        <div class="header-logo">
          <div class="logo-icon">💳</div>
          <div class="logo-text">银行卡支付</div>
        </div>
        <button class="close-button" @click="handleCancel">×</button>
      </div>

      <!-- 步骤指示器 -->
      <div class="step-indicator">
        <div class="step" :class="{ active: currentStep >= 1, completed: currentStep > 1 }">
          <div class="step-number">1</div>
          <div class="step-label">确认信息</div>
        </div>
        <!-- 3DS验证步骤（仅在3DS场景显示） -->
        <template v-if="props.paymentScenario === '3ds_verification'">
          <div 
            class="step-line" 
            :class="{ active: currentStep > 1 }"
          ></div>
          <div 
            class="step" 
            :class="{ 
              active: currentStep >= 2, 
              completed: currentStep > 2
            }"
          >
          <div class="step-number">2</div>
          <div class="step-label">安全验证</div>
        </div>
        </template>
        <!-- 正常流程的连接线 -->
        <div 
          v-else
          class="step-line" 
          :class="{ active: currentStep > 1 }"
        ></div>
        <div class="step" :class="{ active: currentStep >= 3 || (currentStep >= 2 && props.paymentScenario !== '3ds_verification'), completed: currentStep > 3 || (currentStep > 2 && props.paymentScenario !== '3ds_verification') }">
          <div class="step-number">{{ props.paymentScenario === '3ds_verification' ? '3' : '2' }}</div>
          <div class="step-label">完成</div>
        </div>
      </div>

      <!-- 步骤1: 确认支付信息 -->
      <div v-if="currentStep === 1" class="payment-content">
        <div class="confirm-section">
          <h2>确认支付信息</h2>
          
          <!-- 订单信息 -->
          <div class="order-summary">
            <div class="summary-item">
              <span class="summary-label">商品名称</span>
              <span class="summary-value">{{ productName }}</span>
            </div>
            <div class="summary-item">
              <span class="summary-label">订单金额</span>
              <span class="summary-value amount">¥{{ productPrice }}</span>
            </div>
            <div class="summary-divider"></div>
            <div class="summary-item total">
              <span class="summary-label">总计</span>
              <span class="summary-value amount">¥{{ productPrice }}</span>
            </div>
          </div>

          <!-- 卡片信息 -->
          <div class="card-info-section">
            <h3>支付卡片</h3>
            <div class="card-display">
              <div class="card-icon">💳</div>
              <div class="card-details">
                <div class="card-number">{{ maskedCardNumber }}</div>
                <div class="card-holder">{{ cardForm.name }}</div>
                <div class="card-expiry">{{ cardForm.expiry }}</div>
              </div>
            </div>
          </div>

          <!-- 支付协议 -->
          <div class="agreement-section">
            <label class="agreement-checkbox">
              <input 
                type="checkbox" 
                v-model="agreedToTerms"
                :disabled="isProcessing"
              />
              <span>我已阅读并同意《支付服务协议》和《用户协议》</span>
            </label>
          </div>

          <button 
            class="payment-button primary" 
            @click="handleConfirm"
            :disabled="!agreedToTerms || isProcessing"
          >
            {{ isProcessing ? '处理中...' : '确认支付' }}
          </button>
        </div>
      </div>

      <!-- 步骤2: 3D安全验证（仅在3DS验证场景显示） -->
      <div v-if="currentStep === 2 && props.paymentScenario === '3ds_verification'" class="payment-content">
        <div class="verification-section">
          <h2>安全验证</h2>
          <p class="verification-desc">为了保障您的账户安全，需要进行3D安全验证</p>
          
          <div class="verification-methods">
            <div 
              class="verification-method" 
              :class="{ active: verificationMethod === 'sms' }"
              @click="selectVerificationMethod('sms')"
            >
              <div class="method-icon">📱</div>
              <div class="method-info">
                <div class="method-name">短信验证</div>
                <div class="method-desc">发送验证码到手机尾号 {{ maskedPhone }}</div>
              </div>
            </div>
            <div 
              class="verification-method" 
              :class="{ active: verificationMethod === 'email' }"
              @click="selectVerificationMethod('email')"
            >
              <div class="method-icon">📧</div>
              <div class="method-info">
                <div class="method-name">邮箱验证</div>
                <div class="method-desc">发送验证码到邮箱</div>
              </div>
            </div>
          </div>

          <div v-if="verificationMethod" class="verification-form">
            <div class="form-group">
              <label>验证码</label>
              <div class="code-input-group">
                <input 
                  type="text" 
                  v-model="verificationCode" 
                  placeholder="请输入验证码"
                  maxlength="6"
                  :disabled="isProcessing || !codeSent"
                />
                <button 
                  class="send-code-button" 
                  @click="handleSendCode"
                  :disabled="isProcessing || codeSent || countdown > 0"
                >
                  {{ countdown > 0 ? `${countdown}秒后重发` : codeSent ? '已发送' : '发送验证码' }}
                </button>
              </div>
            </div>
            <button 
              class="payment-button primary" 
              @click="handleVerify"
              :disabled="!verificationCode || verificationCode.length !== 6 || isProcessing"
            >
              {{ isProcessing ? '验证中...' : '确认验证' }}
            </button>
          </div>
        </div>
      </div>

      <!-- 步骤3: 支付处理/结果 -->
      <div v-if="currentStep === 3" class="payment-content">
        <div class="payment-result">
          <div v-if="paymentStatus === 'processing'" class="result-processing">
            <div class="spinner-large"></div>
            <h2>正在处理您的支付...</h2>
            <p>请稍候，不要关闭此页面</p>
          </div>
          
          <div v-else-if="paymentStatus === 'success'" class="result-success">
            <div class="result-icon success">✓</div>
            <h2>支付成功！</h2>
            <p>您的订单已成功支付</p>
            <button class="payment-button primary" @click="handleReturn">完成</button>
          </div>
          
          <div v-else-if="paymentStatus === 'error'" class="result-error">
            <div class="result-icon error">✗</div>
            <h2>支付失败</h2>
            <p>{{ errorMessage }}</p>
            <div class="error-actions">
              <button class="payment-button secondary" @click="handleRetry">重试</button>
              <button class="payment-button primary" @click="handleReturn">返回</button>
            </div>
          </div>
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
    default: '游戏道具'
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
  }
})

const emit = defineEmits(['close', 'payment-success', 'payment-error'])

const currentStep = ref(1) // 1: 确认信息, 2: 安全验证, 3: 完成
const paymentStatus = ref('') // '', 'processing', 'success', 'error'
const errorMessage = ref('')
const verificationMethod = ref('')
const verificationCode = ref('')
const codeSent = ref(false)
const countdown = ref(0)
const agreedToTerms = ref(false)
const orderId = ref('')
const paymentTime = ref('')

const isProcessing = computed(() => paymentStatus.value === 'processing')

const maskedCardNumber = computed(() => {
  const number = props.cardForm.number.replace(/\s/g, '')
  if (number.length >= 4) {
    const last4 = number.slice(-4)
    return '**** **** **** ' + last4
  }
  return '**** **** **** ****'
})

const maskedPhone = computed(() => {
  return '1388'
})

// 监听visible变化，重置状态
watch(() => props.visible, (newVal) => {
  if (newVal) {
    resetState()
  }
})

const resetState = () => {
  currentStep.value = 1
  paymentStatus.value = ''
  errorMessage.value = ''
  verificationMethod.value = ''
  verificationCode.value = ''
  codeSent.value = false
  countdown.value = 0
  agreedToTerms.value = false
  orderId.value = generateOrderId()
  paymentTime.value = ''
}

const generateOrderId = () => {
  return 'CARD' + Date.now().toString().slice(-10)
}

const formatPaymentTime = () => {
  const now = new Date()
  const year = now.getFullYear()
  const month = String(now.getMonth() + 1).padStart(2, '0')
  const day = String(now.getDate()).padStart(2, '0')
  const hours = String(now.getHours()).padStart(2, '0')
  const minutes = String(now.getMinutes()).padStart(2, '0')
  return `${year}-${month}-${day} ${hours}:${minutes}`
}

const handleCancel = () => {
  if (!isProcessing.value) {
    emit('close')
  }
}

const handleConfirm = () => {
  if (!agreedToTerms.value || isProcessing.value) return
  
  // 根据paymentScenario决定是否进入3DS验证
  if (props.paymentScenario === '3ds_verification') {
    // 3DS验证场景：进入安全验证步骤
  currentStep.value = 2
  // 默认选择短信验证
  if (!verificationMethod.value) {
    verificationMethod.value = 'sms'
    }
  } else {
    // 正常流程：跳过3DS验证，直接进入支付处理
    currentStep.value = 3
    paymentStatus.value = 'processing'
    
    // 模拟支付处理
    simulatePayment().then(() => {
      if (props.paymentScenario === 'normal') {
        paymentStatus.value = 'success'
        paymentTime.value = formatPaymentTime()
        setTimeout(() => {
          emit('payment-success')
          setTimeout(() => {
            handleReturn()
          }, 7000)
        }, 3000)
      } else {
        handlePaymentError()
      }
    }).catch(() => {
      handlePaymentError()
    })
  }
}

const selectVerificationMethod = (method) => {
  if (!isProcessing.value) {
    verificationMethod.value = method
    verificationCode.value = ''
    codeSent.value = false
  }
}

const handleSendCode = async () => {
  if (isProcessing.value || codeSent.value || countdown.value > 0) return
  
  // 模拟发送验证码
  codeSent.value = true
  countdown.value = 60
  
  const timer = setInterval(() => {
    countdown.value--
    if (countdown.value <= 0) {
      clearInterval(timer)
    }
  }, 1000)
  
  // 模拟自动填入验证码（演示用）
  setTimeout(() => {
    if (verificationCode.value === '') {
      verificationCode.value = '123456'
    }
  }, 1000)
}

const handleVerify = async () => {
  if (!verificationCode.value || verificationCode.value.length !== 6 || isProcessing.value) return
  
  // 模拟验证码验证
  await new Promise(resolve => setTimeout(resolve, 1000))
  
  // 验证成功，进入支付处理
  currentStep.value = 3
  paymentStatus.value = 'processing'
  
  // 模拟支付处理
  try {
    await simulatePayment()
    
    if (props.paymentScenario === 'normal') {
      // 正常流程
      paymentStatus.value = 'success'
      paymentTime.value = formatPaymentTime()
      setTimeout(() => {
        emit('payment-success')
        // 支付成功后自动关闭（延迟关闭以便用户看到成功提示）
        setTimeout(() => {
          handleReturn()
        }, 3000)
      }, 2000)
    } else {
      // 异常场景
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
        reject(new Error('支付超时'))
      } else {
        resolve()
      }
    }, 2500)
  })
}

const handlePaymentError = () => {
  paymentStatus.value = 'error'
  
  switch (props.paymentScenario) {
    case 'card_error':
      errorMessage.value = '银行卡支付失败，请检查卡号或余额'
      break
    case 'network_error':
      errorMessage.value = '网络连接失败，请检查网络后重试'
      break
    case 'timeout':
      errorMessage.value = '支付超时，请稍后重试'
      break
    default:
      errorMessage.value = '支付失败，请稍后重试'
  }
  
  setTimeout(() => {
    emit('payment-error', errorMessage.value)
    }, 3000)
}

const handleRetry = () => {
  currentStep.value = 1
  paymentStatus.value = ''
  errorMessage.value = ''
  verificationCode.value = ''
  codeSent.value = false
  countdown.value = 0
  agreedToTerms.value = false
}

const handleReturn = () => {
  emit('close')
}
</script>

<style scoped>
.card-payment-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
  animation: fadeIn 0.3s ease;
}

.card-payment-container {
  background-color: #ffffff;
  border-radius: 12px;
  width: 90%;
  max-width: 600px;
  max-height: 90vh;
  display: flex;
  flex-direction: column;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.2);
  animation: slideUp 0.3s ease;
  overflow: hidden;
}

.payment-header {
  padding: 20px 24px;
  border-bottom: 1px solid #e0e0e0;
  display: flex;
  justify-content: space-between;
  align-items: center;
  background-color: #fafafa;
}

.header-logo {
  display: flex;
  align-items: center;
  gap: 12px;
}

.logo-icon {
  font-size: 24px;
}

.logo-text {
  font-size: 20px;
  font-weight: 600;
  color: #333;
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
  border-radius: 50%;
  transition: all 0.2s;
}

.close-button:hover {
  background-color: #f5f5f5;
  color: #333;
}

.step-indicator {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 20px 24px;
  background-color: #fafafa;
  border-bottom: 1px solid #e0e0e0;
}

.step {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
}

.step-number {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 14px;
  font-weight: 600;
  background-color: #e0e0e0;
  color: #999;
  transition: all 0.3s;
}

.step.active .step-number {
  background-color: #333;
  color: #ffffff;
}

.step.completed .step-number {
  background-color: #28a745;
  color: #ffffff;
}

.step-label {
  font-size: 12px;
  color: #666;
}

.step.active .step-label {
  color: #333;
  font-weight: 500;
}

.step.hidden {
  display: none;
}

.step-line.hidden {
  display: none;
}

.step-line {
  flex: 1;
  height: 2px;
  background-color: #e0e0e0;
  min-width: 40px;
  margin: 0 12px;
  margin-top: -20px;
}

.step-line.active {
  background-color: #28a745;
}

.payment-content {
  flex: 1;
  padding: 32px 24px;
  overflow-y: auto;
}

.confirm-section h2,
.verification-section h2 {
  font-size: 24px;
  font-weight: 600;
  color: #333;
  margin-bottom: 24px;
  text-align: center;
}

.order-summary {
  background-color: #f5f5f5;
  border-radius: 8px;
  padding: 20px;
  margin-bottom: 24px;
}

.summary-item {
  display: flex;
  justify-content: space-between;
  margin-bottom: 12px;
}

.summary-item.total {
  margin-top: 12px;
  padding-top: 12px;
  border-top: 2px solid #e0e0e0;
  font-weight: 600;
}

.summary-label {
  color: #666;
  font-size: 14px;
}

.summary-value {
  color: #333;
  font-size: 14px;
  font-weight: 500;
}

.summary-value.amount {
  font-size: 18px;
  font-weight: 600;
}

.summary-divider {
  height: 1px;
  background-color: #e0e0e0;
  margin: 12px 0;
}

.card-info-section {
  margin-bottom: 24px;
}

.card-info-section h3 {
  font-size: 16px;
  font-weight: 600;
  color: #333;
  margin-bottom: 12px;
}

.card-display {
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 16px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border-radius: 8px;
  color: #ffffff;
}

.card-icon {
  font-size: 32px;
}

.card-details {
  flex: 1;
}

.card-number {
  font-size: 18px;
  font-weight: 600;
  letter-spacing: 2px;
  margin-bottom: 8px;
}

.card-holder {
  font-size: 14px;
  opacity: 0.9;
  margin-bottom: 4px;
}

.card-expiry {
  font-size: 12px;
  opacity: 0.8;
}

.agreement-section {
  margin-bottom: 24px;
}

.agreement-checkbox {
  display: flex;
  align-items: center;
  gap: 8px;
  font-size: 14px;
  color: #666;
  cursor: pointer;
}

.agreement-checkbox input[type="checkbox"] {
  width: 18px;
  height: 18px;
  cursor: pointer;
}

.verification-desc {
  text-align: center;
  color: #666;
  font-size: 14px;
  margin-bottom: 24px;
}

.verification-methods {
  display: flex;
  flex-direction: column;
  gap: 12px;
  margin-bottom: 24px;
}

.verification-method {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 16px;
  border: 2px solid #e0e0e0;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.2s;
  background-color: #ffffff;
}

.verification-method:hover {
  border-color: #333;
}

.verification-method.active {
  border-color: #333;
  background-color: #f5f5f5;
}

.method-icon {
  font-size: 24px;
  width: 40px;
  height: 40px;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #f5f5f5;
  border-radius: 50%;
}

.method-info {
  flex: 1;
}

.method-name {
  font-size: 14px;
  font-weight: 500;
  color: #333;
  margin-bottom: 4px;
}

.method-desc {
  font-size: 12px;
  color: #999;
}

.verification-form {
  margin-top: 24px;
}

.form-group {
  margin-bottom: 20px;
}

.form-group label {
  display: block;
  font-size: 14px;
  color: #333;
  margin-bottom: 8px;
  font-weight: 500;
}

.code-input-group {
  display: flex;
  gap: 12px;
}

.code-input-group input {
  flex: 1;
  padding: 12px;
  border: 1px solid #e0e0e0;
  border-radius: 6px;
  font-size: 14px;
  color: #333;
  transition: border-color 0.2s;
}

.code-input-group input:focus {
  outline: none;
  border-color: #333;
}

.code-input-group input:disabled {
  background-color: #f5f5f5;
  cursor: not-allowed;
}

.send-code-button {
  padding: 12px 20px;
  background-color: #f5f5f5;
  border: 1px solid #e0e0e0;
  border-radius: 6px;
  font-size: 14px;
  color: #333;
  cursor: pointer;
  transition: all 0.2s;
  white-space: nowrap;
}

.send-code-button:hover:not(:disabled) {
  background-color: #e0e0e0;
}

.send-code-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.payment-button {
  width: 100%;
  padding: 14px 24px;
  border: none;
  border-radius: 6px;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.payment-button.primary {
  background-color: #333;
  color: #ffffff;
}

.payment-button.primary:hover:not(:disabled) {
  background-color: #555;
}

.payment-button.secondary {
  background-color: #f5f5f5;
  color: #333;
  border: 1px solid #e0e0e0;
}

.payment-button.secondary:hover:not(:disabled) {
  background-color: #e0e0e0;
}

.payment-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.payment-result {
  text-align: center;
  padding: 40px 20px;
}

.result-processing,
.result-success,
.result-error {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16px;
}

.spinner-large {
  width: 64px;
  height: 64px;
  border: 4px solid #e0e0e0;
  border-top-color: #333;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

.result-icon {
  width: 80px;
  height: 80px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 48px;
  font-weight: 600;
}

.result-icon.success {
  background-color: #e8f5e9;
  color: #2e7d32;
}

.result-icon.error {
  background-color: #ffebee;
  color: #c62828;
}

.payment-result h2 {
  font-size: 24px;
  font-weight: 600;
  color: #333;
  margin: 0;
}

.payment-result p {
  font-size: 14px;
  color: #666;
  margin: 0;
}

.order-details {
  background-color: #f5f5f5;
  border-radius: 8px;
  padding: 16px;
  margin: 24px 0;
  text-align: left;
  max-width: 400px;
  margin-left: auto;
  margin-right: auto;
}

.detail-item {
  display: flex;
  justify-content: space-between;
  margin-bottom: 8px;
  font-size: 14px;
}

.detail-item:last-child {
  margin-bottom: 0;
}

.detail-item span:first-child {
  color: #666;
}

.detail-item span:last-child {
  color: #333;
  font-weight: 500;
}

.error-actions {
  display: flex;
  gap: 12px;
  margin-top: 24px;
  width: 100%;
  max-width: 400px;
  margin-left: auto;
  margin-right: auto;
}

.error-actions .payment-button {
  flex: 1;
}

@keyframes fadeIn {
  from {
    opacity: 0;
  }
  to {
    opacity: 1;
  }
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

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}
</style>

