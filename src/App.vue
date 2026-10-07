<template>
  <div class="app-container">
    <button
      v-if="!showScenarioPanel"
      class="scenario-panel-trigger"
      type="button"
      @click="showScenarioPanel = true"
    >
      场景模拟
    </button>
    <div
      v-if="showScenarioPanel"
      class="scenario-panel-backdrop"
      @click="showScenarioPanel = false"
    ></div>
    <!-- 左侧调试区 -->
    <aside v-if="showScenarioPanel" class="debug-panel scenario-drawer">
      <div class="debug-header">
        <h2>调试区</h2>
        <button
          class="scenario-panel-close"
          type="button"
          aria-label="关闭场景模拟"
          @click="showScenarioPanel = false"
        >×</button>
      </div>
      <div class="debug-content">
        <!-- 支付场景选择 -->
        <div class="debug-section">
          <h3 class="section-title">支付场景模拟</h3>
          <div class="scenario-grid">
            <!-- 第一排：正常流程和特殊流程 -->
            <div class="scenario-row">
            <div 
              class="scenario-item" 
              :class="{ active: paymentScenario === 'normal' }"
              @click="selectScenario('normal')"
            >
              <div class="scenario-icon">✓</div>
              <div class="scenario-info">
                <div class="scenario-name">正常流程</div>
                <div class="scenario-desc">支付成功场景</div>
              </div>
            </div>
            <div 
              class="scenario-item" 
                :class="{ active: paymentScenario === '3ds_verification' }"
                @click="selectScenario('3ds_verification')"
            >
                <div class="scenario-icon">🔒</div>
              <div class="scenario-info">
                  <div class="scenario-name">3DS验证</div>
                  <div class="scenario-desc">触发3D Secure验证流程</div>
              </div>
            </div>
            <div 
              class="scenario-item" 
              :class="{ active: paymentScenario === 'saved_card' }"
              @click="selectScenario('saved_card')"
            >
              <div class="scenario-icon">💾</div>
              <div class="scenario-info">
                <div class="scenario-name">免 CVV 快速支付</div>
                <div class="scenario-desc">使用已保存的卡一键支付</div>
              </div>
            </div>
            <div 
              class="scenario-item" 
              :class="{ active: paymentScenario === 'saved_card_with_cvv' }"
              @click="selectScenario('saved_card_with_cvv')"
            >
              <div class="scenario-icon">💾</div>
              <div class="scenario-info">
                <div class="scenario-name">已保存卡需 CVV</div>
                <div class="scenario-desc">使用已保存的卡，但需输入 CVV</div>
              </div>
            </div>
            <div 
              class="scenario-item" 
                :class="{ active: paymentScenario === 'timeout' }"
                @click="selectScenario('timeout')"
            >
                <div class="scenario-icon">⏱</div>
              <div class="scenario-info">
                  <div class="scenario-name">支付超时</div>
                  <div class="scenario-desc">订单超时关闭场景</div>
                </div>
              </div>
            </div>

            <!-- 第二排：支付失败场景 -->
            <div class="scenario-row">
              <div class="scenario-category-title">支付失败场景</div>
              <div 
                class="scenario-item" 
                :class="{ active: paymentScenario === 'insufficient_funds' }"
                @click="selectScenario('insufficient_funds')"
              >
                <div class="scenario-icon">💸</div>
                <div class="scenario-info">
                  <div class="scenario-name">资金不足</div>
                  <div class="scenario-desc">卡内余额不足导致支付失败</div>
                </div>
              </div>
              <div 
                class="scenario-item" 
                :class="{ active: paymentScenario === 'card_info_error' }"
                @click="selectScenario('card_info_error')"
              >
                <div class="scenario-icon">💳</div>
                <div class="scenario-info">
                  <div class="scenario-name">卡片信息错误</div>
                  <div class="scenario-desc">卡号/有效期/CVV 填写错误</div>
                </div>
              </div>
              <div 
                class="scenario-item" 
                :class="{ active: paymentScenario === 'issuer_declined' }"
                @click="selectScenario('issuer_declined')"
              >
                <div class="scenario-icon">🏦</div>
                <div class="scenario-info">
                  <div class="scenario-name">发卡行拒绝</div>
                  <div class="scenario-desc">银行拒绝本次交易</div>
              </div>
            </div>
            <div 
              class="scenario-item" 
              :class="{ active: paymentScenario === 'network_error' }"
              @click="selectScenario('network_error')"
            >
              <div class="scenario-icon">⚠</div>
              <div class="scenario-info">
                <div class="scenario-name">网络连接失败</div>
                <div class="scenario-desc">网络异常场景</div>
              </div>
            </div>
            <div 
              class="scenario-item" 
                :class="{ active: paymentScenario === 'generic_error' }"
                @click="selectScenario('generic_error')"
            >
              <div class="scenario-icon">⚠️</div>
              <div class="scenario-info">
                <div class="scenario-name">通用支付失败</div>
                <div class="scenario-desc">兜底支付失败弹窗模版</div>
              </div>
            </div>
            </div>

            <!-- 第三排：账单信息收集场景 -->
            <div class="scenario-row">
              <div class="scenario-category-title">账单信息收集场景</div>
            <div 
              class="scenario-item" 
                :class="{ active: paymentScenario === 'collect_zip_code' }"
                @click="selectScenario('collect_zip_code')"
            >
                <div class="scenario-icon">📮</div>
              <div class="scenario-info">
                  <div class="scenario-name">收集邮编信息</div>
                  <div class="scenario-desc">收银台收集用户邮编信息</div>
              </div>
              </div>
            </div>
          </div>
        </div>

        <div class="debug-section">
          <h3 class="section-title">用户 Country Code</h3>
          <div class="country-code-options">
            <button
              type="button"
              :class="{ active: userCountryCode === 'US' }"
              @click="setUserCountryCode('US')"
            >
              US · 美国
            </button>
            <button
              type="button"
              :class="{ active: userCountryCode === 'CA' }"
              @click="setUserCountryCode('CA')"
            >
              CA · 加拿大
            </button>
          </div>
        </div>

        <!-- Mock卡信息 -->
        <div class="debug-section">
          <h3 class="section-title">Mock 卡信息</h3>
          <div class="mock-card-list">
            <div 
              class="mock-card-item" 
              @click="fillMockCard('mastercard')"
            >
              <div class="card-brand">Mastercard</div>
              <div class="card-number">5555 5555 5555 4444</div>
              <div class="card-info">正常支付</div>
              </div>
            <div 
              class="mock-card-item error" 
              @click="fillMockCard('error')"
            >
              <div class="card-brand">错误卡号</div>
              <div class="card-number">4000 0000 0000 0002</div>
              <div class="card-info">支付失败</div>
            </div>
          </div>
        </div>

        <!-- 当前状态显示 -->
        <div class="debug-section">
          <h3 class="section-title">当前状态</h3>
          <div class="status-info">
            <div class="status-item">
              <span class="status-label">支付场景：</span>
              <span class="status-value">{{ getScenarioName(paymentScenario) }}</span>
            </div>
            <div class="status-item">
              <span class="status-label">收银台状态：</span>
              <span class="status-value">{{ checkoutVisible ? '已打开' : '已关闭' }}</span>
            </div>
          </div>
        </div>

        <!-- 操作日志 -->
        <div class="debug-section">
          <h3 class="section-title">操作日志</h3>
          <div class="log-container">
            <div v-if="logs.length === 0" class="log-empty">暂无日志</div>
            <div v-for="(log, index) in logs" :key="index" class="log-item">
              <span class="log-time">{{ log.time }}</span>
              <span class="log-message">{{ log.message }}</span>
            </div>
          </div>
          <button class="clear-log-button" @click="clearLogs">清空日志</button>
        </div>
      </div>
    </aside>

    <!-- 中间游戏界面demo -->
    <main class="game-demo">
      <div v-if="!checkoutVisible" class="game-container">
        <div class="game-header">
          <h1>游戏界面 Demo</h1>
        </div>
        <div class="game-content">
          <!-- 商品元素，用于触发收银台 -->
          <div class="product-item" @click="handleProductClick">
            <div class="product-icon">🎮</div>
            <div class="product-info">
              <div class="product-name">游戏道具</div>
              <div class="product-price">¥9.99</div>
            </div>
            <div class="product-action">
              <button class="buy-button">购买</button>
            </div>
          </div>
        </div>
      </div>
      <!-- 收银台挂载在 demo 中间区域 -->
      <div class="checkout-anchor">
        <Checkout
          :visible="checkoutVisible"
          :product-name="productName"
          :product-price="productPrice"
          :payment-scenario="paymentScenario"
          :country-code="userCountryCode"
          :initial-card-data="mockCardData"
          @close="handleCheckoutClose"
          @payment-success="handlePaymentSuccess"
          @payment-error="handlePaymentError"
          @jump-to-card="handleJumpToCard"
          @open-3ds="handleOpen3DS"
          @open-paypal="handleOpenPayPal"
        />
      </div>
    </main>

    <!-- 右侧交互说明 -->
    <aside class="help-panel design-hidden-panel">
      <div class="help-header">
        <h2>交互说明</h2>
      </div>
      <div class="help-content">
        <section class="help-section">
          <h3 class="help-section-title">打开收银台</h3>
          <p class="help-text">在中间游戏区域点击「购买」按钮或商品卡片，会弹出收银台浮层。</p>
        </section>
        <section class="help-section">
          <h3 class="help-section-title">支付方式</h3>
          <p class="help-text">收银台支持<strong>信用卡/借记卡</strong>与<strong>PayPal</strong>。点击对应选项切换，选中的方式会高亮显示。</p>
        </section>
        <section class="help-section">
          <h3 class="help-section-title">银行卡支付</h3>
          <ul class="help-list">
            <li>填写持卡人姓名、卡号、有效期(MM/YY)、CVV；卡号会自动按 4 位空格格式化。</li>
            <li>支持 VISA、Mastercard、AMEX、Discover；输入卡号后可自动识别卡品牌。</li>
            <li>若左侧选择了「免 CVV 快速支付」或「已保存卡需 CVV」，会显示已保存卡列表，可选中后一键支付或仅输入 CVV。</li>
            <li>可勾选「保存此卡信息」以便下次使用（演示环境）。</li>
          </ul>
        </section>
        <section class="help-section">
          <h3 class="help-section-title">账单信息</h3>
          <p class="help-text">选择「收集邮编」时，右侧会显示邮编输入区域；填写有效邮编后会计算并显示税费与总计。</p>
        </section>
        <section class="help-section">
          <h3 class="help-section-title">协议与提交</h3>
          <p class="help-text">需勾选「我已阅读并同意服务协议和隐私政策」方可点击「去支付」。点击「去支付」后会进行表单校验，通过则进入支付处理或 3DS/PayPal 流程。</p>
        </section>
        <section class="help-section">
          <h3 class="help-section-title">3DS 与 PayPal</h3>
          <p class="help-text">选择 3DS 验证场景时，点击「去支付」会先打开 3D Secure 验证页，完成验证后关闭验证页并继续支付。选择 PayPal 时会打开 PayPal 支付页，完成或取消后返回收银台。</p>
        </section>
        <section class="help-section">
          <h3 class="help-section-title">关闭与取消</h3>
          <p class="help-text">点击收银台右上角「✕」或点击遮罩，若尚未支付会弹出「确认要取消支付吗？」；选择「继续支付」留在收银台，选择「确认取消」关闭收银台。</p>
        </section>
        <section class="help-section">
          <h3 class="help-section-title">支付结果</h3>
          <p class="help-text">支付成功会显示「支付成功！」弹窗，可点击「完成」或等待倒计时后关闭。支付失败会显示具体原因，可点击「返回重新支付」回到表单。订单超时场景下点击「去支付」会提示「订单已超时」并需重新下单。</p>
        </section>
      </div>
    </aside>

    <!-- 3DS 验证 Webview -->
    <div v-if="threeDSWebviewVisible" class="webview-overlay">
      <div class="webview-container">
        <div class="webview-header">
          <div class="webview-header-left">
            <div class="webview-nav-buttons">
              <button class="nav-btn back-btn" title="后退">‹</button>
              <button class="nav-btn forward-btn" title="前进">›</button>
              <button class="nav-btn refresh-btn" title="刷新">↻</button>
            </div>
            <span class="webview-title">3D Secure 验证</span>
          </div>
          <div class="webview-header-right">
            <button class="webview-close-btn" @click="handle3DSWebviewClose" title="关闭">✕</button>
          </div>
        </div>
        <div class="webview-content">
          <div class="three-ds-webview-content">
            <div class="third-party-highlight">
              <div class="third-party-badge">第三方验证页面</div>
              <p class="third-party-text">
                您即将跳转至 <strong>发卡行/银行卡组织的 3D Secure 验证页面</strong> 完成安全验证。
              </p>
              <p class="third-party-text">
                本页面仅为演示环境中的模拟界面，<strong>不代表商户自建的实际验证页面</strong>。
              </p>
            </div>
            <button
              class="result-button primary"
              @click="handle3DSVerify"
              :disabled="is3DSProcessing"
            >
              <span v-if="is3DSProcessing" class="button-spinner"></span>
              <span>{{ is3DSProcessing ? '验证中...' : '我已在银行页面完成验证' }}</span>
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- PayPal 支付 Webview -->
    <PayPalPayment
      :visible="paypalWebviewVisible"
      :product-name="paypalWebviewData.productName"
      :product-price="paypalWebviewData.productPrice"
      :payment-scenario="paypalWebviewData.paymentScenario"
      :inline-mode="false"
      @close="handlePayPalWebviewClose"
      @payment-success="handlePayPalWebviewSuccess"
      @payment-error="handlePayPalWebviewError"
    />

    <!-- 卡支付页面 -->
    <CardPayment
      :visible="cardVisible"
      :product-name="productName"
      :product-price="productPrice"
      :payment-scenario="paymentScenario"
      :card-form="cardFormData"
      @close="handleCardClose"
      @payment-success="handleCardSuccess"
      @payment-error="handleCardError"
    />
  </div>
</template>

<script setup>
import { ref } from 'vue'
import Checkout from './components/Checkout.vue'
import CardPayment from './components/CardPayment.vue'
import PayPalPayment from './components/PayPalPayment.vue'

const checkoutVisible = ref(true)
const cardVisible = ref(false)
const showScenarioPanel = ref(false)
const paymentScenario = ref('normal')
const userCountryCode = ref('US')
const productName = ref('预谋的纸质书签')
const productPrice = ref('88')
const cardFormData = ref({
  name: '',
  number: '',
  expiry: '',
  cvv: ''
})
const mockCardData = ref(null) // Mock卡信息数据
const logs = ref([])

// 3DS 和 PayPal Webview 状态
const threeDSWebviewVisible = ref(false)
const paypalWebviewVisible = ref(false)
const is3DSProcessing = ref(false)
const paypalWebviewData = ref({
  productName: '',
  productPrice: '',
  paymentScenario: '',
})

// Mock卡信息配置
const mockCards = {
  mastercard: {
    name: 'Jane Smith',
    number: '5555 5555 5555 4444',
    expiry: '06/26',
    cvv: '456'
  },
  error: {
    name: 'Error Card',
    number: '4000 0000 0000 0002',
    expiry: '12/25',
    cvv: '123'
  }
}

// 填充Mock卡信息
const fillMockCard = (cardType) => {
  if (mockCards[cardType]) {
    mockCardData.value = { ...mockCards[cardType] }
    addLog(`填充Mock卡信息：${cardType}`)
    
    // 如果选择错误卡号，自动切换到card_error场景
    if (cardType === 'error') {
      paymentScenario.value = 'card_error'
      addLog('自动切换到：银行卡支付失败场景')
    }
    
    // 如果收银台未打开，先打开它
    if (!checkoutVisible.value) {
      checkoutVisible.value = true
      addLog('打开收银台')
    }
  }
}

const getScenarioName = (scenario) => {
  const names = {
    normal: '正常流程',
    card_error: '银行卡支付失败',
    insufficient_funds: '资金不足',
    card_info_error: '卡片信息错误',
    issuer_declined: '发卡行拒绝',
    three_ds_failed: '3D Secure 验证失败',
    risk_blocked: '风控拦截',
    paypal_error: 'PayPal 支付失败',
    network_error: '网络连接失败',
    timeout: '订单超时关闭',
    '3ds_verification': '3DS验证',
    'saved_card': '免 CVV 快速支付（仅演示）',
    'saved_card_with_cvv': '已保存卡需 CVV',
    'generic_error': '通用支付失败',
    'collect_zip_code': '收集邮编信息'
  }
  return names[scenario] || '未知'
}

const addLog = (message) => {
  const now = new Date()
  const time = `${now.getHours().toString().padStart(2, '0')}:${now.getMinutes().toString().padStart(2, '0')}:${now.getSeconds().toString().padStart(2, '0')}`
  logs.value.unshift({ time, message })
  if (logs.value.length > 50) {
    logs.value = logs.value.slice(0, 50)
  }
}

const clearLogs = () => {
  logs.value = []
  addLog('日志已清空')
}

const selectScenario = (scenario) => {
  paymentScenario.value = scenario
  checkoutVisible.value = true
  showScenarioPanel.value = false
  addLog(`切换支付场景：${getScenarioName(scenario)}`)
}

const setUserCountryCode = (countryCode) => {
  userCountryCode.value = countryCode
  addLog(`用户 Country Code：${countryCode}`)
}

const handleProductClick = () => {
  checkoutVisible.value = true
  addLog('打开收银台')
}

const handleCheckoutClose = () => {
  checkoutVisible.value = false
  mockCardData.value = null // 关闭时清空mock数据
  addLog('关闭收银台')
}

// 打开 3DS 验证 Webview
const handleOpen3DS = (data) => {
  threeDSWebviewVisible.value = true
  addLog('打开 3DS 验证 Webview')
}

// 关闭 3DS 验证 Webview
const handle3DSWebviewClose = () => {
  threeDSWebviewVisible.value = false
  is3DSProcessing.value = false
  addLog('关闭 3DS 验证 Webview')
}

// 3DS 验证完成
const handle3DSVerify = async () => {
  is3DSProcessing.value = true
  addLog('3DS 验证中...')
  
  // 模拟验证过程
  await new Promise((resolve) => setTimeout(resolve, 1500))
  
  is3DSProcessing.value = false
  threeDSWebviewVisible.value = false
  
  // 验证成功后，打开收银台并继续支付流程
  addLog('3DS 验证成功，继续支付')
  checkoutVisible.value = true
  
  // 触发事件，让 Checkout 组件继续支付流程
  // 使用 nextTick 确保收银台已经打开
  setTimeout(() => {
    window.dispatchEvent(new CustomEvent('continue-3ds-payment'))
  }, 100)
}

// 打开 PayPal 支付 Webview
const handleOpenPayPal = (data) => {
  paypalWebviewData.value = {
    productName: data.productName || productName.value,
    productPrice: data.productPrice || productPrice.value,
    paymentScenario: data.paymentScenario || paymentScenario.value,
  }
  paypalWebviewVisible.value = true
  addLog('打开 PayPal 支付 Webview')
}

// 关闭 PayPal 支付 Webview
const handlePayPalWebviewClose = (reason) => {
  paypalWebviewVisible.value = false
  if (reason === 'back-to-checkout') {
    // 支付失败返回收银台
    checkoutVisible.value = true
    addLog('PayPal 支付失败，返回收银台')
  } else if (reason === 'cancel-to-checkout') {
    // 用户取消支付，返回收银台并显示取消交易弹窗
    checkoutVisible.value = true
    addLog('PayPal 支付取消，返回收银台')
    // 触发事件，让 Checkout 组件显示取消交易弹窗
    setTimeout(() => {
      window.dispatchEvent(new CustomEvent('show-paypal-cancel-dialog'))
    }, 100)
  } else if (reason === 'success-to-game') {
    // 支付成功返回游戏
    addLog('PayPal 支付成功，返回游戏界面')
  } else {
    addLog('关闭 PayPal 支付 Webview')
}
}

// PayPal 支付成功
const handlePayPalWebviewSuccess = () => {
  paypalWebviewVisible.value = false
  handlePaymentSuccess({
    orderId: 'PP' + Date.now().toString().slice(-10),
    amount: paypalWebviewData.value.productPrice,
    paymentMethod: 'paypal',
    redirectToGame: true,
  })
}

// PayPal 支付失败
const handlePayPalWebviewError = (errorMessage) => {
  addLog('PayPal 支付失败: ' + (errorMessage || '未知错误'))
}

const handlePaymentSuccess = (paymentData) => {
  // 支付结果页由收银台内部控制显示时长和关闭时机（用户点击"完成"）
  addLog('支付成功')
  
  // 如果是 PayPal 支付成功，重定向到游戏界面
  if (paymentData && paymentData.paymentMethod === 'paypal' && paymentData.redirectToGame) {
  checkoutVisible.value = false
    addLog('PayPal 支付成功，重定向到游戏界面')
    // 可以在这里添加其他重定向逻辑，比如滚动到游戏界面等
    setTimeout(() => {
      const gameDemo = document.querySelector('.game-demo')
      if (gameDemo) {
        gameDemo.scrollIntoView({ behavior: 'smooth', block: 'start' })
      }
    }, 100)
  }
}

const handlePaymentError = (errorMessage) => {
  addLog(`支付失败：${errorMessage}`)
}


const handleJumpToCard = (cardData) => {
  checkoutVisible.value = false
  cardFormData.value = cardData
  cardVisible.value = true
  addLog('跳转到银行卡支付页面')
}

const handleCardClose = () => {
  cardVisible.value = false
  addLog('关闭银行卡支付页面')
}

const handleCardSuccess = () => {
  addLog('银行卡支付成功')
  cardVisible.value = false
}

const handleCardError = (errorMessage) => {
  addLog(`银行卡支付失败：${errorMessage}`)
}
</script>

<style scoped>
.app-container {
  display: flex;
  width: 100vw;
  height: 100vh;
  background-color: #f5f5f5;
}

.scenario-panel-trigger {
  position: fixed;
  top: 76px;
  left: 0;
  z-index: 12000;
  padding: 10px 12px;
  border: 0;
  border-radius: 0 8px 8px 0;
  color: #fff;
  background: #252525;
  box-shadow: 0 4px 14px rgba(0, 0, 0, 0.18);
  font-size: 13px;
  cursor: pointer;
}

.scenario-panel-trigger:hover {
  color: #f6c914;
}

.scenario-panel-backdrop {
  position: fixed;
  inset: 0;
  z-index: 12001;
  background: rgba(0, 0, 0, 0.25);
}

.scenario-drawer {
  position: fixed;
  inset: 0 auto 0 0;
  z-index: 12002;
  width: min(430px, 92vw);
  min-width: 0;
  box-shadow: 8px 0 28px rgba(0, 0, 0, 0.22);
}

.scenario-panel-close {
  width: 30px;
  height: 30px;
  border: 0;
  color: #333;
  background: transparent;
  font-size: 24px;
  cursor: pointer;
}

.scenario-drawer .scenario-row {
  grid-template-columns: repeat(2, minmax(0, 1fr));
}

.scenario-drawer .scenario-item {
  min-width: 0;
}

.scenario-drawer .scenario-name,
.scenario-drawer .scenario-desc {
  overflow-wrap: anywhere;
}

.country-code-options {
  display: grid;
  grid-template-columns: repeat(2, minmax(0, 1fr));
  gap: 8px;
}

.country-code-options button {
  padding: 10px 8px;
  border: 1px solid #d9dde3;
  border-radius: 7px;
  color: #444;
  background: #fff;
  font-size: 13px;
  cursor: pointer;
}

.country-code-options button.active {
  border-color: #f1bb00;
  background: #fffbef;
  color: #222;
  font-weight: 600;
}

/* 左侧调试区 */
.debug-panel {
  width: 25%;
  min-width: 300px;
  background-color: #ffffff;
  border-right: 1px solid #e0e0e0;
  display: flex;
  flex-direction: column;
  box-shadow: 2px 0 4px rgba(0, 0, 0, 0.05);
}

.debug-header {
  padding: 20px;
  border-bottom: 1px solid #e0e0e0;
  background-color: #fafafa;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.debug-header h2 {
  font-size: 18px;
  font-weight: 600;
  color: #333;
}

.debug-content {
  flex: 1;
  padding: 20px;
  overflow-y: auto;
}

.debug-section {
  margin-bottom: 24px;
}

.debug-section:last-child {
  margin-bottom: 0;
}

.section-title {
  font-size: 14px;
  font-weight: 600;
  color: #333;
  margin-bottom: 12px;
  padding-bottom: 8px;
  border-bottom: 1px solid #e0e0e0;
}

.scenario-grid {
  display: flex;
  flex-direction: column;
  gap: 16px;
}

.scenario-row {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 8px;
}

.scenario-category-title {
  grid-column: 1 / -1;
  font-size: 12px;
  font-weight: 600;
  color: #666;
  margin-bottom: 4px;
  padding-bottom: 6px;
  border-bottom: 1px solid #e0e0e0;
}

.scenario-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.scenario-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px;
  border: 2px solid #e0e0e0;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s;
  background-color: #ffffff;
}

.scenario-item:hover {
  border-color: #999;
  background-color: #fafafa;
}

.scenario-item.active {
  border-color: #333;
  background-color: #f5f5f5;
}

.scenario-icon {
  width: 24px;
  height: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 16px;
  flex-shrink: 0;
}

.scenario-info {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.scenario-name {
  font-size: 14px;
  font-weight: 500;
  color: #333;
}

.scenario-desc {
  font-size: 12px;
  color: #999;
}

.status-info {
  background-color: #f5f5f5;
  border-radius: 6px;
  padding: 12px;
}

.status-item {
  display: flex;
  justify-content: space-between;
  margin-bottom: 8px;
}

.status-item:last-child {
  margin-bottom: 0;
}

.status-label {
  font-size: 13px;
  color: #666;
}

.status-value {
  font-size: 13px;
  color: #333;
  font-weight: 500;
}

.log-container {
  background-color: #f5f5f5;
  border-radius: 6px;
  padding: 12px;
  max-height: 200px;
  overflow-y: auto;
  margin-bottom: 12px;
  font-family: 'Monaco', 'Menlo', 'Courier New', monospace;
}

.log-empty {
  font-size: 12px;
  color: #999;
  text-align: center;
  padding: 20px 0;
}

.log-item {
  display: flex;
  gap: 8px;
  margin-bottom: 6px;
  font-size: 11px;
  line-height: 1.4;
}

.log-item:last-child {
  margin-bottom: 0;
}

.log-time {
  color: #999;
  flex-shrink: 0;
}

.log-message {
  color: #333;
  word-break: break-word;
}

.clear-log-button {
  width: 100%;
  padding: 8px;
  background-color: #f5f5f5;
  border: 1px solid #e0e0e0;
  border-radius: 6px;
  font-size: 12px;
  color: #666;
  cursor: pointer;
  transition: all 0.2s;
}

.clear-log-button:hover {
  background-color: #e0e0e0;
  color: #333;
}

/* Mock卡信息 */
.mock-card-list {
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.mock-card-item {
  padding: 12px;
  border: 2px solid #e0e0e0;
  border-radius: 6px;
  cursor: pointer;
  transition: all 0.2s;
  background-color: #ffffff;
}

.mock-card-item:hover {
  border-color: #3b82f6;
  background-color: #eff6ff;
  transform: translateX(4px);
}

.mock-card-item.error {
  border-color: #ef4444;
}

.mock-card-item.error:hover {
  border-color: #dc2626;
  background-color: #fef2f2;
}

.card-brand {
  font-size: 13px;
  font-weight: 600;
  color: #333;
  margin-bottom: 4px;
}

.mock-card-item.error .card-brand {
  color: #dc2626;
}

.card-number {
  font-size: 12px;
  font-family: 'Monaco', 'Menlo', 'Courier New', monospace;
  color: #666;
  margin-bottom: 4px;
  letter-spacing: 0.5px;
}

.card-info {
  font-size: 11px;
  color: #999;
}

/* 中间游戏界面demo */
.game-demo {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #f5f5f5;
  padding: 40px;
  position: relative;
  min-width: 0;
}

/* 收银台挂载在 demo 中间，覆盖中间区域 */
.checkout-anchor {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  pointer-events: none;
}

.checkout-anchor > * {
  pointer-events: auto;
}

.game-container {
  width: 100%;
  max-width: 1200px;
  height: 100%;
  background-color: #ffffff;
  border: 1px solid #e0e0e0;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.game-header {
  padding: 20px 30px;
  border-bottom: 1px solid #e0e0e0;
  background-color: #fafafa;
}

.game-header h1 {
  font-size: 24px;
  font-weight: 600;
  color: #333;
}

.game-content {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 40px;
}

/* 右侧交互说明 */
.help-panel {
  width: 26%;
  min-width: 280px;
  max-width: 380px;
  background-color: #ffffff;
  border-left: 1px solid #e0e0e0;
  display: flex;
  flex-direction: column;
  box-shadow: -2px 0 4px rgba(0, 0, 0, 0.05);
}

.help-header {
  padding: 20px;
  border-bottom: 1px solid #e0e0e0;
  background-color: #fafafa;
}

.help-header h2 {
  font-size: 18px;
  font-weight: 600;
  color: #333;
  margin: 0;
}

.help-content {
  flex: 1;
  padding: 20px;
  overflow-y: auto;
}

.help-section {
  margin-bottom: 20px;
}

.help-section:last-child {
  margin-bottom: 0;
}

.help-section-title {
  font-size: 14px;
  font-weight: 600;
  color: #333;
  margin: 0 0 8px 0;
  padding-bottom: 6px;
  border-bottom: 1px solid #e5e7eb;
}

.help-text {
  font-size: 13px;
  color: #555;
  line-height: 1.6;
  margin: 0;
}

.help-text strong {
  color: #333;
  font-weight: 600;
}

.help-list {
  margin: 0;
  padding-left: 18px;
  font-size: 13px;
  color: #555;
  line-height: 1.65;
}

.help-list li {
  margin-bottom: 6px;
}

.help-list li:last-child {
  margin-bottom: 0;
}

/* 商品元素 */
.product-item {
  display: flex;
  align-items: center;
  gap: 20px;
  padding: 24px 32px;
  background-color: #ffffff;
  border: 2px solid #e0e0e0;
  border-radius: 8px;
  cursor: pointer;
  transition: all 0.3s ease;
  min-width: 400px;
  box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
}

.product-item:hover {
  border-color: #999;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  transform: translateY(-2px);
}

.product-icon {
  font-size: 48px;
  width: 80px;
  height: 80px;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #f5f5f5;
  border-radius: 8px;
  border: 1px solid #e0e0e0;
}

.product-info {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 8px;
}

.product-name {
  font-size: 20px;
  font-weight: 600;
  color: #333;
}

.product-price {
  font-size: 18px;
  color: #666;
  font-weight: 500;
}

.product-action {
  display: flex;
  align-items: center;
}

.buy-button {
  padding: 12px 32px;
  background-color: #333;
  color: #ffffff;
  border: none;
  border-radius: 6px;
  font-size: 16px;
  font-weight: 500;
  cursor: pointer;
  transition: background-color 0.3s ease;
}

.buy-button:hover {
  background-color: #555;
}

.buy-button:active {
  background-color: #222;
}

/* Webview 样式 */
.webview-overlay {
  position: fixed;
  top: 0;
  left: 25%; /* 从调试区右侧开始，只覆盖游戏界面区域 */
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  z-index: 10000;
  display: flex;
  align-items: center;
  justify-content: center;
}

.webview-container {
  width: 90vw;
  max-width: 1200px;
  height: 90vh;
  background-color: #ffffff;
  border-radius: 12px;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  display: flex;
  flex-direction: column;
  overflow: hidden;
}

.webview-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  height: 44px;
  padding: 0 16px;
  background: linear-gradient(90deg, #111827 0%, #1f2937 60%, #111827 100%);
  color: #f9fafb;
  border-bottom: 1px solid rgba(31, 41, 55, 0.8);
}

.webview-header-left {
  display: flex;
  align-items: center;
  gap: 12px;
}

.webview-nav-buttons {
  display: flex;
  align-items: center;
  gap: 4px;
}

.webview-nav-buttons .nav-btn {
  width: 28px;
  height: 28px;
  border: none;
  background-color: rgba(248, 250, 252, 0.08);
  border-radius: 6px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 16px;
  color: #e5e7eb;
  transition: background-color 0.15s ease, color 0.15s ease;
}

.webview-nav-buttons .nav-btn:hover {
  background-color: rgba(248, 250, 252, 0.15);
  color: #ffffff;
}

.webview-title {
  font-size: 14px;
  font-weight: 600;
}

.webview-header-right {
  display: flex;
  align-items: center;
}

.webview-close-btn {
  border: none;
  width: 24px;
  height: 24px;
  border-radius: 999px;
  background-color: rgba(248, 250, 252, 0.08);
  color: #e5e7eb;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  font-size: 14px;
  transition: background-color 0.15s ease, color 0.15s ease, transform 0.1s ease;
}

.webview-close-btn:hover {
  background-color: #ef4444;
  color: #ffffff;
  transform: scale(1.05);
}

.webview-content {
  flex: 1;
  overflow: auto;
  background-color: #ffffff;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 40px;
}

.three-ds-webview-content {
  max-width: 600px;
  width: 100%;
  text-align: center;
}

.third-party-highlight {
  background-color: #fef3c7;
  border: 2px solid #fbbf24;
  border-radius: 12px;
  padding: 24px;
  margin-bottom: 24px;
}

.third-party-badge {
  display: inline-block;
  background-color: #f59e0b;
  color: #ffffff;
  padding: 6px 12px;
  border-radius: 6px;
  font-size: 12px;
  font-weight: 600;
  margin-bottom: 12px;
}

.third-party-text {
  font-size: 14px;
  color: #92400e;
  line-height: 1.6;
  margin: 8px 0;
}

.result-button {
  padding: 12px 24px;
  border-radius: 8px;
  border: none;
  font-size: 15px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.15s;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
}

.result-button.primary {
  background-color: #2563eb;
  color: #ffffff;
}

.result-button.primary:hover:not(:disabled) {
  background-color: #1d4ed8;
}

.result-button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.button-spinner {
  width: 16px;
  height: 16px;
  border: 2px solid rgba(255, 255, 255, 0.3);
  border-top-color: #ffffff;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
  flex-shrink: 0;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

.design-hidden-panel {
  display: none !important;
}
</style>
