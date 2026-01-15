<template>
  <div v-if="visible" class="checkout-overlay" @click.self="handleOverlayClick">
    <div class="checkout-modal">
      <!-- 收银台内容区域 -->
      <div class="checkout-content">
        <!-- 收银台头部（关闭按钮） -->
      <div class="checkout-header">
          <button class="checkout-close-btn" @click="handleClose" title="关闭">✕</button>
      </div>

        <!-- 支付表单页面 -->
        <div class="checkout-body" :class="{ 'blurred': currentStep === 'success' || currentStep === 'error' }">
        <div class="checkout-page-header">
          <h2 class="checkout-page-title">收银台</h2>
          <p class="checkout-page-subtitle">完成本次支付以获取商品</p>
          </div>
        <!-- 页面中间的加载spinner -->
        <div v-if="isProcessing" class="page-loading-overlay">
          <div class="page-loading-container">
            <div class="page-loading-spinner"></div>
          </div>
        </div>
        <div class="checkout-layout">
          <!-- 左侧：支付方式和卡信息 -->
          <div class="checkout-left">
        <!-- 支付方式选择 -->
        <div class="payment-methods">
              <h3>支付方式</h3>
          <div class="method-options">
            <div 
              class="method-option" 
              :class="{ active: paymentMethod === 'card' }"
              @click="selectPaymentMethod('card')"
            >
              <div class="method-icon">💳</div>
                  <div class="method-name">信用卡 / 借记卡</div>
                  <div v-if="paymentMethod === 'card'" class="method-check">✓</div>
            </div>
            <div 
              class="method-option" 
              :class="{ active: paymentMethod === 'paypal' }"
              @click="selectPaymentMethod('paypal')"
            >
              <div class="method-icon">🔵</div>
              <div class="method-name">PayPal</div>
                  <div v-if="paymentMethod === 'paypal'" class="method-check">✓</div>
                </div>
              </div>

              <!-- 支持的卡片品牌 -->
              <div v-if="paymentMethod === 'card'" class="card-brands">
                <span class="brand-label">支持的卡片：</span>
                <div class="brand-icons">
                  <span class="brand-icon">VISA</span>
                  <span class="brand-icon">Mastercard</span>
                  <span class="brand-icon">AMEX</span>
                  <span class="brand-icon">Discover</span>
            </div>
          </div>
        </div>

            <!-- 已保存卡片（仅在 saved_card / saved_card_with_cvv 场景），参考 Stripe/Adyen 的保存卡交互 -->
            <div
              v-if="paymentMethod === 'card' && (paymentScenario === 'saved_card' || paymentScenario === 'saved_card_with_cvv') && savedCards.length > 0"
              class="saved-cards-section"
            >
              <h4 class="saved-cards-title">
                <span v-if="paymentScenario === 'saved_card'">已保存的卡片（免 CVV 快速支付，仅演示）</span>
                <span v-else>选择一张已保存的卡完成支付</span>
              </h4>
              <div class="saved-cards-list">
                <div
                  v-for="(card, index) in savedCards"
                  :key="card.id"
                  class="saved-card-option"
                  :class="{ active: selectedSavedCardIndex === index }"
                >
                  <div class="saved-card-row" @click="selectSavedCard(index)">
                    <span
                      class="saved-card-radio"
                      :class="{ active: selectedSavedCardIndex === index }"
                    ></span>
                    <span class="saved-card-brand-text">{{ card.brand }}</span>
                    <span class="saved-card-mask">{{ card.maskedNumber }}</span>
                    <button
                      class="saved-card-delete"
                      type="button"
                      title="删除已保存卡"
                      @click.stop="handleDeleteSavedCard(index)"
                      :disabled="isProcessing"
                    >
                      <svg viewBox="0 0 20 20" aria-hidden="true">
                        <path
                          d="M6 6h8l-.6 9a2 2 0 0 1-2 1.9H8.6a2 2 0 0 1-2-1.9L6 6Zm2.5-3h3a1 1 0 0 1 1 1v1H7.5V4a1 1 0 0 1 1-1ZM4 5h12"
                          fill="none"
                          stroke="currentColor"
                          stroke-width="1.6"
                          stroke-linecap="round"
                          stroke-linejoin="round"
                        />
                      </svg>
                    </button>
                  </div>
                  <!-- CVV 输入框：仅在 saved_card_with_cvv 场景且当前卡片被选中时显示 -->
                  <div
                    v-if="
                      paymentScenario === 'saved_card_with_cvv' &&
                      selectedSavedCardIndex === index
                    "
                    class="saved-card-cvv-section"
                  >
                    <label class="cvv-label">Security code</label>
                    <div class="cvv-input-wrapper" :class="{ 'has-error': cvvError }">
                      <input
                        type="password"
                        v-model="cardForm.cvv"
                        :maxlength="selectedCardBrand === 'American Express' ? 4 : 3"
                        @focus="cvvFocused = true"
                        @blur="cvvFocused = false; validateCVV()"
                        placeholder=""
                        class="cvv-input-field"
                        :disabled="isProcessing"
                      />
                      <span class="cvv-card-icon" aria-hidden="true">
                        <svg viewBox="0 0 32 24">
                          <rect x="1" y="3" width="30" height="18" rx="4" fill="#eef2f7" stroke="#cbd5e1"/>
                          <rect x="1" y="7" width="30" height="4" fill="#cbd5e1"/>
                          <rect x="6" y="14" width="12" height="5" rx="2" fill="#ffffff" stroke="#e2e8f0"/>
                          <rect x="20" y="14" width="6" height="5" rx="2" fill="#ffffff" stroke="#ef4444"/>
                        </svg>
                      </span>
                    </div>
                    <p class="cvv-helper-text">
                      {{ selectedCardBrand === 'American Express' ? '4 digits on front of card' : '3 digits on back of card' }}
                    </p>
                    <p v-if="cvvError" class="error-message cvv-error-message">
                      {{ cvvError }}
                    </p>
                  </div>
                </div>
                <!-- 使用新卡 -->
                <div
                  class="saved-card-option add-new-card-option"
                  :class="{ active: selectedSavedCardIndex === -1 }"
                  @click="selectNewCard"
                >
                  <div class="saved-card-row add-new-card-row">
                    <span
                      class="saved-card-radio"
                      :class="{ active: selectedSavedCardIndex === -1 }"
                    ></span>
                    <span class="saved-card-add-text">使用新卡添加并支付</span>
                    <span class="saved-card-add-icon">+</span>
                  </div>
                </div>
              </div>
            </div>

            <!-- 已保存卡片说明文案 -->
            <p
              v-if="paymentMethod === 'card' && (paymentScenario === 'saved_card' || paymentScenario === 'saved_card_with_cvv') && savedCards.length > 0"
              class="saved-card-tip"
            >
              <span v-if="paymentScenario === 'saved_card'">
                已保存的卡信息将直接用于本次支付（演示环境，免 CVV）。
              </span>
              <span v-else>
                出于安全考虑，我们仅保存卡号等基础信息，本次支付仍需输入安全码（CVV）。
              </span>
            </p>


            <!-- 卡支付表单
                 - 正常/失败/账单信息等场景：始终展示完整表单
                 - saved_card 场景：仅在未选择已保存卡或选择“使用新卡”时展示
                 - saved_card_with_cvv 场景：当未选择已保存卡或选择“使用新卡”时展示完整表单；
                   选择已保存卡时只在卡片下方展示精简 CVV 输入（参考 Stripe/Adyen） -->
            <div
              v-if="
                paymentMethod === 'card' &&
                (
                  paymentScenario !== 'saved_card' &&
                  paymentScenario !== 'saved_card_with_cvv'
                ||
                  (paymentScenario === 'saved_card' && !selectedSavedCard) ||
                  (paymentScenario === 'saved_card_with_cvv' && !selectedSavedCard)
                )
              "
              class="payment-form"
            >
              <!-- 持卡人姓名 -->
              <div
                class="form-group"
                :class="{ 'has-error': cardNameError, focused: cardNameFocused }"
              >
            <label>持卡人姓名</label>
            <input 
              type="text" 
              v-model="cardForm.name" 
                  @focus="cardNameFocused = true"
                  @blur="cardNameFocused = false; validateCardName()"
              placeholder="请输入持卡人姓名"
              :disabled="isProcessing"
            />
                <div v-if="cardNameError" class="error-message">
                  {{ cardNameError }}
          </div>
              </div>

              <!-- 卡号 -->
              <div
                class="form-group"
                :class="{ 'has-error': cardNumberError, focused: cardNumberFocused }"
              >
            <label>卡号</label>
                <div class="card-number-row">
            <input 
              type="text" 
              v-model="cardForm.number" 
              @input="formatCardNumber"
                    @focus="cardNumberFocused = true"
                    @blur="cardNumberFocused = false; validateCardNumber()"
              placeholder="1234 5678 9012 3456"
                    maxlength="23"
              :disabled="isProcessing"
            />
                  <div v-if="detectedCardBrand" class="card-brand-badge">
                    {{ detectedCardBrand }}
          </div>
                </div>
                <div v-if="cardNumberError" class="error-message">
                  {{ cardNumberError }}
                </div>
              </div>

              <!-- 有效期 + CVV -->
          <div class="form-row">
                <div
                  class="form-group half"
                  :class="{ 'has-error': expiryError, focused: expiryFocused }"
                >
              <label>有效期</label>
              <input 
                type="text" 
                v-model="cardForm.expiry" 
                @input="formatExpiry"
                    @focus="expiryFocused = true"
                    @blur="expiryFocused = false; validateExpiry()"
                placeholder="MM/YY"
                maxlength="5"
                :disabled="isProcessing"
              />
                  <div v-if="expiryError" class="error-message">
                    {{ expiryError }}
            </div>
                </div>
                <div
                  class="form-group half"
                  :class="{ 'has-error': cvvError, focused: cvvFocused }"
                >
              <label>CVV</label>
              <input 
                type="text" 
                v-model="cardForm.cvv" 
                @input="formatCVV"
                    @focus="cvvFocused = true"
                    @blur="cvvFocused = false; validateCVV()"
                    placeholder="3-4位数字"
                    maxlength="4"
                :disabled="isProcessing"
              />
                  <div v-if="cvvError" class="error-message">
                    {{ cvvError }}
                  </div>
            </div>
          </div>

              <!-- 保存卡信息 -->
          <div class="save-card-section">
            <label class="save-card-checkbox">
              <input 
                type="checkbox" 
                v-model="saveCard"
                :disabled="isProcessing"
              />
              <span class="save-card-text">
                    <svg
                      width="16"
                      height="16"
                      viewBox="0 0 16 16"
                      fill="none"
                      class="save-card-icon"
                    >
                      <rect
                        x="1.5"
                        y="3.5"
                        width="13"
                        height="9"
                        rx="1.5"
                        stroke="currentColor"
                        stroke-width="1.2"
                      />
                      <path
                        d="M2 5h12"
                        stroke="currentColor"
                        stroke-width="1.2"
                        stroke-linecap="round"
                      />
                      <rect
                        x="4"
                        y="8"
                        width="4"
                        height="2"
                        rx="0.5"
                        fill="currentColor"
                      />
                </svg>
                    保存此卡信息，方便下次快速支付
              </span>
            </label>
                <p class="save-card-desc">
                  为保障安全，我们不会直接存储完整卡号，只会存储经过加密的
                  token。
                </p>
          </div>
        </div>
      </div>

          <!-- 右侧：订单摘要 & 支付按钮 -->
          <div class="checkout-right">
            <div class="order-summary">
              <h3 class="summary-title">订单摘要</h3>
              <div class="summary-content">
                <div class="summary-item">
                  <span class="summary-label">商品</span>
                  <span class="summary-value">{{ productName }}</span>
                </div>
                <div class="summary-item">
                  <span class="summary-label">价格</span>
                  <span class="summary-value">¥{{ productPrice }}</span>
                </div>
                <div v-if="showTax" class="summary-item">
                  <span class="summary-label">税费</span>
                  <span class="summary-value">
                    {{ taxAmount > 0 ? `¥${taxAmount.toFixed(2)}` : '-' }}
                  </span>
                </div>
                <div class="summary-divider"></div>
                <div class="summary-item total">
                  <span class="summary-label">总计</span>
                  <span class="summary-value total-amount">
                    US${{ totalAmount.toFixed(2) }}
                  </span>
                </div>
              </div>
            </div>

            <!-- 账单信息摘要（根据场景显示） -->
            <div v-if="showBillingInfo" class="billing-summary">
              <h3 class="summary-title">账单信息</h3>
              <div class="billing-summary-content">
                <div class="billing-summary-item">
                  <span class="billing-label">国家</span>
                  <input
                    type="text"
                    value="United States"
                    class="billing-input readonly"
                    readonly
                    disabled
                  />
                </div>
                <div 
                  v-if="showZipCode"
                  class="billing-summary-item"
                  :class="{ 'has-error': zipCodeError }"
                >
                  <span class="billing-label">邮编*</span>
                  <input 
                    type="text" 
                    v-model="billingForm.zipCode" 
                    @focus="zipCodeFocused = true"
                    @blur="zipCodeFocused = false; validateZipCode()"
                    placeholder="ZIP Code"
                    class="billing-input"
                    :disabled="isProcessing"
                  />
                  <div v-if="zipCodeError" class="billing-error-message">
                    {{ zipCodeError }}
                  </div>
                </div>
                <div 
                  v-if="showEmail"
                  class="billing-summary-item email-input-wrapper"
                  :class="{ 'has-error': emailError, 'is-focused': emailFocused }"
                >
                  <label class="billing-label email-label">
                    <span>邮箱（可选）</span>
                    <span class="email-hint">用于接收收据</span>
                  </label>
                  <div class="email-input-container">
                    <svg 
                      class="email-icon" 
                      width="16" 
                      height="16" 
                      viewBox="0 0 16 16" 
                      fill="none"
                    >
                      <path 
                        d="M2 4L8 8L14 4M2 4H14M2 4V12H14V4" 
                        stroke="currentColor" 
                        stroke-width="1.5" 
                        stroke-linecap="round" 
                        stroke-linejoin="round"
                      />
                    </svg>
                    <input 
                      type="email" 
                      v-model="billingForm.email" 
                      @focus="emailFocused = true"
                      @blur="emailFocused = false; validateEmail()"
                      placeholder="Enter email for receipt"
                      class="billing-input email-input"
                      :disabled="isProcessing"
                    />
                  </div>
                  <div v-if="emailError" class="billing-error-message email-error">
                    {{ emailError }}
                  </div>
                </div>
                <div class="save-billing-summary">
                  <label class="save-billing-checkbox">
                    <input 
                      type="checkbox" 
                      v-model="saveBillingInfo"
                      :disabled="isProcessing"
                    />
                    <span class="save-billing-text">保存账单信息</span>
                  </label>
                </div>
              </div>
            </div>

            <!-- 协议 -->
      <div class="terms-section">
        <label class="terms-checkbox">
          <input 
            type="checkbox" 
            v-model="agreedToTerms"
            :disabled="isProcessing"
          />
          <span class="terms-text">
            我已阅读并同意
                  <a href="#" class="terms-link" @click.prevent="openTerms">
                    服务协议
                  </a>
            和
                  <a href="#" class="terms-link" @click.prevent="openPrivacy">
                    隐私政策
                  </a>
          </span>
        </label>
      </div>

            <!-- 支付按钮 -->
      <div class="checkout-footer">
              <!-- saved_card 场景 + 选中已保存卡：一键支付 -->
        <button 
                v-if="
                  paymentMethod === 'card' &&
                  paymentScenario === 'saved_card' &&
                  selectedSavedCard &&
                  canQuickPay
                "
                class="confirm-button"
                @click="handleQuickPayment"
                :disabled="isProcessing || !canQuickPay"
              >
                <span v-if="isProcessing" class="button-spinner"></span>
                <span>{{ isProcessing ? '处理中...' : '去支付' }}</span>
              </button>
              <!-- 其他场景 -->
              <button
                v-else
          class="confirm-button" 
          @click="handleConfirmPayment"
          :disabled="isProcessing || !canSubmit"
        >
                <span v-if="isProcessing" class="button-spinner"></span>
                <span>{{ isProcessing ? '处理中...' : '去支付' }}</span>
        </button>
            </div>
          </div>
        </div>
      </div>

      <!-- 支付结果悬浮弹窗（覆盖整个收银台） -->
      <div
        v-if="
          currentStep === 'success' ||
          currentStep === 'error'
        "
        class="payment-result-overlay"
      >
        <div class="payment-result-modal">
          <div class="payment-result-content">
            <!-- 支付成功 -->
            <div v-if="currentStep === 'success'" class="result-success">
              <div class="result-icon success">✓</div>
              <h2>支付成功！</h2>
              <p>您的订单已成功支付。</p>
              <div class="order-details">
                <div class="detail-item">
                  <span>订单号：</span>
                  <span>{{ orderId }}</span>
                </div>
                <div class="detail-item">
                  <span>支付金额：</span>
                  <span>¥{{ productPrice }}</span>
                </div>
                <div class="detail-item">
                  <span>支付方式：</span>
                  <span v-if="currentPaymentMethod === 'card'">银行卡 ({{ maskedCardNumber }})</span>
                  <span v-else-if="currentPaymentMethod === 'paypal'">PayPal</span>
                </div>
                <div class="detail-item">
                  <span>支付时间：</span>
                  <span>{{ paymentTime }}</span>
                </div>
              </div>
              <button class="result-button primary" @click="handlePaymentComplete">
                完成
              </button>
            </div>

            <!-- 支付失败 -->
            <div v-else-if="currentStep === 'error'" class="result-error">
              <div class="result-icon error">✗</div>
              <h2>支付失败</h2>
              <p>{{ errorMessage }}</p>
              <div class="error-actions">
                <button class="result-button primary" @click="handleReturnToForm">
                  返回重新支付
                </button>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- 通用支付失败弹窗（兜底模版，覆盖整个收银台窗口） -->
      <div
        v-if="showGenericErrorDialog"
        class="payment-result-overlay"
        @click.self="showGenericErrorDialog = false"
      >
        <div class="payment-result-modal">
          <div class="payment-result-content">
            <div class="result-error">
              <div class="result-icon error">✗</div>
              <h2>支付失败</h2>
              <p>很抱歉，我们无法处理您的支付。请稍后再试或尝试其他支付方式。</p>
              <div class="error-actions">
                <button class="result-button primary" @click="handleGenericErrorConfirm">
                  我知道了
                </button>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- 支付超时弹窗 -->
      <div
        v-if="showExpiredDialog"
        class="payment-result-overlay"
        @click.self="showExpiredDialog = false"
      >
        <div class="payment-result-modal">
          <div class="payment-result-content">
            <div class="result-error">
              <div class="result-icon error">⚠</div>
              <h2>订单已超时</h2>
              <p>该订单因长时间未支付已自动关闭，无法继续支付，请重新下单。</p>
              <div class="error-actions">
                <button class="result-button primary" @click="handleExpiredConfirm">
                  确认
                </button>
              </div>
            </div>
          </div>
        </div>
      </div>

      <!-- PayPal 取消交易弹窗 -->
      <div
        v-if="showPayPalCancelDialog"
        class="payment-result-overlay"
        @click.self="showPayPalCancelDialog = false"
      >
        <div class="payment-result-modal">
          <div class="payment-result-content">
            <div class="result-error">
              <div class="result-icon error">ℹ</div>
              <h2>交易已取消</h2>
              <p>您已取消本次 PayPal 支付，已返回收银台页面。</p>
              <div class="error-actions">
                <button class="result-button primary" @click="handlePayPalCancelConfirm">
                  我知道了
                </button>
              </div>
            </div>
          </div>
        </div>
      </div>
      </div>

      <!-- 取消支付确认对话框 -->
      <div
        v-if="showCancelDialog"
        class="cancel-dialog-overlay"
        @click.self="showCancelDialog = false"
      >
        <div class="cancel-dialog">
          <h3 class="cancel-title">确认要取消支付吗？</h3>
          <p class="cancel-desc">
            订单尚未完成支付，取消后可能需要重新下单。
          </p>
          <div class="cancel-actions">
            <button class="cancel-btn secondary" @click="handleContinuePayment">
              继续支付
            </button>
            <button class="cancel-btn primary" @click="handleConfirmCancel">
              确认取消
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { computed, ref, watch, onMounted, onUnmounted } from 'vue'

const props = defineProps({
  visible: {
    type: Boolean,
    default: false,
  },
  productName: {
    type: String,
    default: '',
  },
  productPrice: {
    type: String,
    default: '0.00',
  },
  paymentScenario: {
    type: String,
    default: 'normal', // normal, card_error, paypal_error, network_error, timeout, 3ds_verification, saved_card
  },
  initialCardData: {
    type: Object,
    default: () => null,
  },
})

const emit = defineEmits([
  'close',
  'payment-success',
  'payment-error',
  'jump-to-card',
  'open-3ds',
  'open-paypal',
])

// 基础状态
const paymentMethod = ref('card')
const paymentStatus = ref('') // '', 'processing', 'success', 'error'
const errorMessage = ref('')
const agreedToTerms = ref(true)
const saveCard = ref(false)
const showCancelDialog = ref(false)
const showExpiredMessage = ref(false)
const showExpiredDialog = ref(false) // 支付超时弹窗
const showGenericErrorDialog = ref(false) // 通用支付失败兜底弹窗
const showPayPalCancelDialog = ref(false) // PayPal 取消交易弹窗
const orderId = ref('')
const paymentTime = ref('')
const currentPaymentMethod = ref('card') // 当前使用的支付方式

// 流程状态
const currentStep = ref('form') // 'form', '3ds_verification', 'processing', 'success', 'error'
const verificationMethod = ref('') // 'sms', 'email'
const verificationCode = ref('')
const codeSent = ref(false)
const countdown = ref(0)

// 已保存卡
const selectedSavedCardIndex = ref(-1)
const savedCards = ref([
  {
    id: '1',
    brand: 'Visa',
    maskedNumber: '•••• •••• •••• 1111',
    fullNumber: '4111 1111 1111 1111',
    name: 'JOHN DOE',
    expiry: '12/25',
    cvv: '123',
  },
  {
    id: '2',
    brand: 'Mastercard',
    maskedNumber: '•••• •••• •••• 4444',
    fullNumber: '5555 5555 5555 4444',
    name: 'JANE SMITH',
    expiry: '06/26',
    cvv: '456',
  },
  {
    id: '3',
    brand: 'American Express',
    maskedNumber: '•••• •••••• •0005',
    fullNumber: '3782 822463 10005',
    name: 'ROBERT JOHNSON',
    expiry: '09/25',
    cvv: '1234',
  },
])

const selectedSavedCard = computed(() => {
  if (
    selectedSavedCardIndex.value >= 0 &&
    savedCards.value[selectedSavedCardIndex.value]
  ) {
    return savedCards.value[selectedSavedCardIndex.value]
  }
  return null
})

// 当前选中卡片的品牌（用于 CVV 提示）
const selectedCardBrand = computed(() => {
  if (selectedSavedCard.value) {
    return selectedSavedCard.value.brand
  }
  return detectedCardBrand.value
})

// 表单状态
const cardNameFocused = ref(false)
const cardNumberFocused = ref(false)
const expiryFocused = ref(false)
const cvvFocused = ref(false)

const cardNameError = ref('')
const cardNumberError = ref('')
const expiryError = ref('')
const cvvError = ref('')

const cardForm = ref({
  name: '',
  number: '',
  expiry: '',
  cvv: '',
})

// 账单信息
const billingForm = ref({
  zipCode: '',
  email: '',
})
const saveBillingInfo = ref(false)
const zipCodeFocused = ref(false)
const zipCodeError = ref('')
const emailFocused = ref(false)
const emailError = ref('')

// 是否显示账单信息字段
const showBillingInfo = computed(() => {
  return props.paymentScenario === 'collect_zip_code' || 
         props.paymentScenario === 'collect_email' ||
         props.paymentScenario === 'collect_zip_and_email'
})

// 是否显示邮编字段
const showZipCode = computed(() => {
  return props.paymentScenario === 'collect_zip_code' ||
         props.paymentScenario === 'collect_zip_and_email'
})

// 是否显示邮箱字段
const showEmail = computed(() => {
  return props.paymentScenario === 'collect_email' ||
         props.paymentScenario === 'collect_zip_and_email'
})

// 税费计算（根据邮编，只在收集邮编场景且邮编填写完成后计算）
const taxAmount = computed(() => {
  if (props.paymentScenario !== 'collect_zip_code' && 
      props.paymentScenario !== 'collect_zip_and_email') {
    return 0
  }
  const zipCode = billingForm.value.zipCode.trim()
  if (!zipCode || zipCode.length < 3) {
    return 0
  }
  // 模拟税费计算：根据邮编计算税费（简单示例：邮编长度 * 0.1）
  // 实际场景中，这里应该调用后端API根据邮编计算税费
  const basePrice = parseFloat(props.productPrice) || 0
  const taxRate = 0.08 // 8% 税费率（示例）
  return basePrice * taxRate
})

// 是否显示税费字段
const showTax = computed(() => {
  return props.paymentScenario === 'collect_zip_code' ||
         props.paymentScenario === 'collect_zip_and_email'
})

// 总金额（包含税费）
const totalAmount = computed(() => {
  const basePrice = parseFloat(props.productPrice) || 0
  if (showTax.value && taxAmount.value > 0) {
    return basePrice + taxAmount.value
  }
  return basePrice
})

// 订单是否已超时
const orderExpired = computed(() => props.paymentScenario === 'timeout')

const isProcessing = computed(() => paymentStatus.value === 'processing')

// 卡品牌识别
const detectedCardBrand = computed(() => {
  const number = cardForm.value.number.replace(/\s/g, '')
  if (number.startsWith('4')) return 'Visa'
  if (number.startsWith('5') || number.startsWith('2')) return 'Mastercard'
  if (number.startsWith('3')) return 'Amex'
  if (number.startsWith('6')) return 'Discover'
  return ''
})

// 掩码卡号
const maskedCardNumber = computed(() => {
  const number = cardForm.value.number.replace(/\s/g, '')
  if (number.length >= 4) {
    const last4 = number.slice(-4)
    return '**** **** **** ' + last4
  }
  return '**** **** **** ****'
})

// 已保存卡一键支付是否可用
const canQuickPay = computed(() => {
  if (!agreedToTerms.value) return false
  if (
    props.paymentScenario === 'saved_card' &&
    paymentMethod.value === 'card' &&
    selectedSavedCard.value
  ) {
    return true
  }
  return false
})

// 普通表单是否可以提交
const canSubmit = computed(() => {
  if (!agreedToTerms.value) return false
  if (paymentMethod.value === 'card') {
    // 参考 Stripe/Adyen：已保存卡 + 仅补 CVV 的场景下，放宽对卡片基础字段的校验，只要求 CVV 有效
    let cardValid
    if (props.paymentScenario === 'saved_card_with_cvv' && selectedSavedCard.value) {
      cardValid = cardForm.value.cvv.length >= 3 && !cvvError.value
    } else {
      cardValid = (
        cardForm.value.name.trim() &&
        cardForm.value.number.replace(/\s/g, '').length >= 13 &&
        cardForm.value.expiry.replace(/\D/g, '').length === 4 &&
        cardForm.value.cvv.length >= 3 &&
        !cardNumberError.value &&
        !cardNameError.value &&
        !expiryError.value &&
        !cvvError.value
      )
    }
    
    // 如果需要收集账单信息，验证账单信息
  if (showBillingInfo.value) {
      const billingValid = (
        (!showZipCode.value || billingForm.value.zipCode.trim()) &&
        !zipCodeError.value &&
        !emailError.value
      )
      return cardValid && billingValid
    }
    
    return cardValid
  }
  return true
})

// 工具函数
const getCardBrandIcon = (brand) => {
  const icons = {
    Visa: 'VISA',
    Mastercard: 'MC',
    'American Express': 'AMEX',
    Discover: 'DISC',
  }
  return icons[brand] || 'CARD'
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

// 监听组件可见性
watch(
  () => props.visible,
  (val) => {
    if (val) {
  resetPaymentStatus()
      agreedToTerms.value = true
      saveCard.value = false
    }
  }
)

// 监听场景切换
watch(
  () => props.paymentScenario,
  (scenario) => {
    resetPaymentStatus()
    if (scenario === 'saved_card' || scenario === 'saved_card_with_cvv') {
      paymentMethod.value = 'card'
      if (savedCards.value.length > 0) {
        selectedSavedCardIndex.value = 0
        // 在需 CVV 场景下，自动用第一张卡预填表单（CVV 留空）
        if (scenario === 'saved_card_with_cvv') {
          const card = savedCards.value[0]
          cardForm.value = {
            name: card.name,
            number: card.fullNumber,
            expiry: card.expiry,
            cvv: '',
          }
        }
      } else {
        selectedSavedCardIndex.value = -1
      }
    } else {
      selectedSavedCardIndex.value = -1
  }
  }
)

// 监听 3DS 验证完成事件，继续支付流程
const handleContinue3DSPayment = () => {
  if (props.paymentScenario === '3ds_verification' && props.visible) {
    // 3DS 验证已完成，继续支付流程
    processPayment()
}
}

// 监听 PayPal 取消交易事件，显示取消弹窗
const handleShowPayPalCancelDialog = () => {
  if (props.visible) {
    showPayPalCancelDialog.value = true
  }
}

// 处理 PayPal 取消交易弹窗确认
const handlePayPalCancelConfirm = () => {
  showPayPalCancelDialog.value = false
    resetPaymentStatus()
}

onMounted(() => {
  window.addEventListener('continue-3ds-payment', handleContinue3DSPayment)
  window.addEventListener('show-paypal-cancel-dialog', handleShowPayPalCancelDialog)
})

onUnmounted(() => {
  window.removeEventListener('continue-3ds-payment', handleContinue3DSPayment)
  window.removeEventListener('show-paypal-cancel-dialog', handleShowPayPalCancelDialog)
})

const resetPaymentStatus = () => {
  paymentStatus.value = ''
  errorMessage.value = ''
  orderId.value = ''
  paymentTime.value = ''
  currentStep.value = 'form'
  verificationMethod.value = ''
  verificationCode.value = ''
  codeSent.value = false
  countdown.value = 0
  currentPaymentMethod.value = 'card'

  // 重置表单
  if (props.initialCardData) {
    cardForm.value = {
      name: props.initialCardData.name || '',
      number: props.initialCardData.number || '',
      expiry: props.initialCardData.expiry || '',
      cvv: props.initialCardData.cvv || '',
    }
  } else {
  cardForm.value = {
    name: '',
    number: '',
    expiry: '',
      cvv: '',
  }
}

  // 重置账单信息
  billingForm.value = {
    zipCode: '',
    email: '',
  }
  saveBillingInfo.value = false

  clearErrors()
  cardNameFocused.value = false
  cardNumberFocused.value = false
  expiryFocused.value = false
  cvvFocused.value = false
  showCancelDialog.value = false
  showExpiredMessage.value = false
}

// 输入格式化 & 校验
const clearErrors = () => {
  cardNameError.value = ''
  cardNumberError.value = ''
  expiryError.value = ''
  cvvError.value = ''
  zipCodeError.value = ''
  emailError.value = ''
}

const formatCardNumber = (event) => {
  let value = event.target.value.replace(/\D/g, '')
  value = value.replace(/(.{4})/g, '$1 ').trim()
  cardForm.value.number = value
  if (cardNumberFocused.value) {
    cardNumberError.value = ''
  }
}

const formatExpiry = (event) => {
  let value = event.target.value.replace(/\D/g, '')
  if (value.length > 4) value = value.slice(0, 4)
  if (value.length >= 3) {
    value = value.slice(0, 2) + '/' + value.slice(2)
  }
  cardForm.value.expiry = value
  if (expiryFocused.value) {
    expiryError.value = ''
  }
}

const formatCVV = (event) => {
  let value = event.target.value.replace(/\D/g, '')
  const maxLength = detectedCardBrand.value === 'Amex' ? 4 : 3
  if (value.length > maxLength) {
    value = value.slice(0, maxLength)
  }
  cardForm.value.cvv = value
  if (cvvFocused.value) {
    cvvError.value = ''
  }
}

// Luhn 算法
const luhnCheck = (num) => {
  let sum = 0
  let shouldDouble = false
  for (let i = num.length - 1; i >= 0; i--) {
    let digit = parseInt(num[i], 10)
    if (shouldDouble) {
      digit *= 2
      if (digit > 9) digit -= 9
    }
    sum += digit
    shouldDouble = !shouldDouble
  }
  return sum % 10 === 0
}

const validateCardNumber = () => {
  const number = cardForm.value.number.replace(/\s/g, '')
  if (!number) {
    cardNumberError.value = '请输入卡号'
    return false
  }
  if (!/^\d+$/.test(number)) {
    cardNumberError.value = '卡号只能包含数字'
    return false
  }
  if (number.length < 13 || number.length > 19) {
    cardNumberError.value = '卡号长度不正确'
    return false
  }
  if (!luhnCheck(number)) {
    cardNumberError.value = '卡号无效，请检查后重新输入'
    return false
  }
  cardNumberError.value = ''
  return true
}

const validateCardName = () => {
  const name = cardForm.value.name.trim()
  if (!name) {
    cardNameError.value = '请输入持卡人姓名'
    return false
  }
  if (name.length < 2) {
    cardNameError.value = '姓名长度过短'
    return false
  }
  if (!/^[a-zA-Z\s\-'.]+$/.test(name)) {
    cardNameError.value = '持卡人姓名只能包含字母、空格、连字符和撇号'
    return false
  }
  cardNameError.value = ''
  return true
}

const validateExpiry = () => {
  const expiry = cardForm.value.expiry.replace(/\D/g, '')
  if (!expiry) {
    expiryError.value = '请输入有效期'
    return false
  }
  if (expiry.length !== 4) {
    expiryError.value = '有效期格式不正确，请输入MM/YY'
    return false
  }
  const month = parseInt(expiry.slice(0, 2), 10)
  const year = parseInt(expiry.slice(2, 4), 10)
  if (month < 1 || month > 12) {
    expiryError.value = '月份不正确'
    return false
  }
  const now = new Date()
  const currentYear = now.getFullYear() % 100
  const currentMonth = now.getMonth() + 1
  if (year < currentYear || (year === currentYear && month < currentMonth)) {
    expiryError.value = '卡片已过期'
    return false
  }
  expiryError.value = ''
  return true
}

const validateCVV = () => {
  const cvv = cardForm.value.cvv
  if (!cvv) {
    cvvError.value = '请输入CVV安全码'
    return false
  }
  const isAmex = selectedCardBrand.value === 'American Express' || detectedCardBrand.value === 'Amex'
  const requiredLength = isAmex ? 4 : 3
  if (cvv.length !== requiredLength) {
    cvvError.value =
      isAmex
        ? 'American Express 的 CVV 为4位'
        : 'CVV 应为3位数字'
    return false
  }
  if (!/^\d+$/.test(cvv)) {
    cvvError.value = 'CVV 只能包含数字'
    return false
  }
  cvvError.value = ''
  return true
}

// 验证邮编
const validateZipCode = () => {
  const zipCode = billingForm.value.zipCode.trim()
  if (!zipCode) {
    zipCodeError.value = '请输入邮编'
    return false
  }
  if (zipCode.length < 3 || zipCode.length > 10) {
    zipCodeError.value = '邮编格式不正确'
    return false
  }
  zipCodeError.value = ''
  return true
}

// 验证邮箱（可选项：为空时不报错，有值时校验格式）
const validateEmail = () => {
  const email = billingForm.value.email.trim()
  // 非必填：如果为空，认为通过校验
  if (!email) {
    emailError.value = ''
    return true
  }
  // 简单的邮箱格式验证（仅在有值时校验）
  const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/
  if (!emailRegex.test(email)) {
    emailError.value = 'Please enter a valid email address'
    return false
  }
  emailError.value = ''
  return true
}

// 验证账单地址
// 操作
const selectPaymentMethod = (method) => {
  if (isProcessing.value) return
  paymentMethod.value = method
}

const handleOverlayClick = () => {
    handleClose()
}

const handleClose = () => {
  if (isProcessing.value) return
  // 如果已展示结果，直接关闭
  if (currentStep.value === 'success' || currentStep.value === 'error') {
    emit('close')
    return
  }
  showCancelDialog.value = true
}

const handleContinuePayment = () => {
  showCancelDialog.value = false
}

const handleConfirmCancel = () => {
  showCancelDialog.value = false
  emit('close')
}

// 核心支付逻辑
const handleConfirmPayment = async () => {
  if (!canSubmit.value || isProcessing.value) return

  if (orderExpired.value) {
    showExpiredDialog.value = true
    return
  }

  if (paymentMethod.value === 'paypal') {
    // 触发打开 PayPal 支付 Webview 事件
    emit('open-paypal', {
      productName: props.productName,
      productPrice: props.productPrice,
      paymentScenario: props.paymentScenario,
    })
    return
  }

  if (paymentMethod.value === 'card') {
    const okNumber = validateCardNumber()
    const okName = validateCardName()
    const okExpiry = validateExpiry()
    const okCVV = validateCVV()
    if (!okNumber || !okName || !okExpiry || !okCVV) return

    // 验证账单信息（如果需要）
    if (showBillingInfo.value) {
      if (showZipCode.value) {
        const okZipCode = validateZipCode()
        if (!okZipCode) return
      }
      if (showEmail.value) {
        const okEmail = validateEmail()
        if (!okEmail) return
      }
    }

    // 3DS 场景触发打开 3DS 验证 Webview 事件
    if (props.paymentScenario === '3ds_verification') {
      emit('open-3ds', {
        productName: props.productName,
        productPrice: props.productPrice,
        paymentScenario: props.paymentScenario,
    })
    return
    }

    await processPayment()
  }
}

// 已保存卡一键支付
const selectSavedCard = (index) => {
  selectedSavedCardIndex.value = index

  // 非免 CVV 场景：选择已保存卡时自动填充表单信息，但保留 CVV 为空，需用户输入
  if (props.paymentScenario === 'saved_card_with_cvv') {
    const card = savedCards.value[index]
    if (card) {
      cardForm.value = {
        name: card.name,
        number: card.fullNumber,
        expiry: card.expiry,
        cvv: '',
      }
      cardNameError.value = ''
      cardNumberError.value = ''
      expiryError.value = ''
      cvvError.value = ''
    }
  }
}

const handleDeleteSavedCard = (index) => {
  if (isProcessing.value) return
  const isSelected = selectedSavedCardIndex.value === index
  savedCards.value.splice(index, 1)

  if (savedCards.value.length === 0) {
    selectedSavedCardIndex.value = -1
    if (props.paymentScenario === 'saved_card_with_cvv') {
      selectNewCard()
    }
    return
  }

  if (isSelected) {
    const nextIndex = Math.min(index, savedCards.value.length - 1)
    selectedSavedCardIndex.value = nextIndex
    if (props.paymentScenario === 'saved_card_with_cvv') {
      const card = savedCards.value[nextIndex]
      if (card) {
        cardForm.value = {
          name: card.name,
          number: card.fullNumber,
          expiry: card.expiry,
          cvv: '',
        }
        cardNameError.value = ''
        cardNumberError.value = ''
        expiryError.value = ''
        cvvError.value = ''
      }
    }
  } else if (selectedSavedCardIndex.value > index) {
    selectedSavedCardIndex.value -= 1
  }
}

const selectNewCard = () => {
  selectedSavedCardIndex.value = -1
  // 清空表单，进入新卡录入流程
  cardForm.value = {
    name: '',
    number: '',
    expiry: '',
    cvv: '',
  }
  clearErrors()
}

const handleQuickPayment = async () => {
  if (!canQuickPay.value || isProcessing.value) return

  if (orderExpired.value) {
    showExpiredDialog.value = true
    return
  }

  const card = selectedSavedCard.value
  if (!card) return

  // 直接填充已保存卡信息（含 CVV），模拟一键支付
  cardForm.value = {
    name: card.name,
    number: card.fullNumber,
    expiry: card.expiry,
    cvv: card.cvv,
  }

  if (props.paymentScenario === '3ds_verification') {
    emit('open-3ds', {
      productName: props.productName,
      productPrice: props.productPrice,
      paymentScenario: props.paymentScenario,
    })
    return
  }

  await processPayment()
}

// 3DS 相关
const selectVerificationMethod = (method) => {
  if (isProcessing.value) return
  verificationMethod.value = method
  verificationCode.value = ''
  codeSent.value = false
  countdown.value = 0
}

const handleSendCode = () => {
  if (isProcessing.value || codeSent.value || countdown.value > 0) return
  codeSent.value = true
  countdown.value = 60
  const timer = setInterval(() => {
    countdown.value--
    if (countdown.value <= 0) {
      clearInterval(timer)
    }
  }, 1000)

  // 演示环境：自动填入验证码
  setTimeout(() => {
    if (!verificationCode.value) {
      verificationCode.value = '123456'
    }
  }, 1000)
}

// 3DS 验证现在由 App.vue 中的独立 Webview 处理
// handleVerify3DS 函数已移除，3DS 验证逻辑在独立 Webview 中处理

// 支付过程
const processPayment = async () => {
  currentPaymentMethod.value = 'card'
  currentStep.value = 'processing'
  paymentStatus.value = 'processing'
  orderId.value = generateOrderId()
  try {
    await simulatePayment()
    if (
      props.paymentScenario === 'normal' ||
      props.paymentScenario === '3ds_verification' ||
      props.paymentScenario === 'saved_card' ||
      props.paymentScenario === 'saved_card_with_cvv'
    ) {
      currentStep.value = 'success'
      paymentStatus.value = 'success'
      paymentTime.value = formatPaymentTime()
      // 不在这里触发 payment-success，等用户点击"完成"按钮时统一处理
    } else {
      handlePaymentError()
    }
  } catch (e) {
    handlePaymentError()
  }
}

const simulatePayment = () => {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (
        props.paymentScenario === 'card_error' ||
        props.paymentScenario === 'network_error' ||
        props.paymentScenario === 'insufficient_funds' ||
        props.paymentScenario === 'card_info_error' ||
        props.paymentScenario === 'issuer_declined' ||
        props.paymentScenario === 'three_ds_failed' ||
        props.paymentScenario === 'risk_blocked' ||
        props.paymentScenario === 'generic_error'
      ) {
        reject(new Error('支付失败'))
      } else {
        resolve()
      }
    }, 1500)
  })
}

const handlePaymentError = () => {
  // 通用支付失败场景特殊处理：先重置状态，避免一直加载中
  if (props.paymentScenario === 'generic_error') {
    // 重置支付状态，避免显示加载中或普通错误结果页
    currentStep.value = 'form'
    paymentStatus.value = ''
    // 显示通用支付失败弹窗
    showGenericErrorDialog.value = true
    // 触发错误事件，用于日志记录
    setTimeout(() => {
      emit('payment-error', '通用支付失败，已显示兜底弹窗')
    }, 100)
    return
  }
  
  // 其他错误场景：显示普通错误结果页
  currentStep.value = 'error'
  paymentStatus.value = 'error'
  switch (props.paymentScenario) {
    case 'card_error':
      errorMessage.value = '银行卡支付失败，请检查卡片信息或余额。'
      break
    case 'insufficient_funds':
      errorMessage.value = '支付失败：资金不足，请更换卡片或充值后重试。'
      break
    case 'card_info_error':
      errorMessage.value = '支付失败：卡片信息错误，请检查卡号、有效期和 CVV。'
      break
    case 'issuer_declined':
      errorMessage.value = '支付失败：发卡行拒绝本次交易，请联系发卡行或更换卡片。'
      break
    case 'three_ds_failed':
      errorMessage.value = '支付失败：3D Secure 验证失败，请重新验证或更换支付方式。'
      break
    case 'risk_blocked':
      errorMessage.value = '支付失败：触发风控拦截，请稍后重试或更换支付方式。'
      break
    case 'paypal_error':
      errorMessage.value = 'PayPal 支付失败，请稍后重试。'
      break
    case 'network_error':
      errorMessage.value = '网络连接失败，请检查网络后重试。'
      break
    case 'timeout':
      errorMessage.value = '订单已超时，请重新下单。'
      break
    default:
      errorMessage.value = '支付失败，请稍后重试。'
  }
  setTimeout(() => {
    emit('payment-error', errorMessage.value)
  }, 500)
}

const handlePaymentComplete = () => {
  // 如果是 PayPal 支付成功，传递支付方式信息以便重定向到游戏界面
  if (currentPaymentMethod.value === 'paypal') {
    emit('payment-success', {
      orderId: orderId.value,
      amount: props.productPrice,
      paymentMethod: 'paypal',
      redirectToGame: true
    })
  } else {
    // 银行卡支付成功，只触发成功事件，不重定向
    emit('payment-success', {
      orderId: orderId.value,
      amount: props.productPrice,
      paymentMethod: 'card',
      redirectToGame: false
    })
  }
  emit('close')
}

// 处理支付超时确认
const handleExpiredConfirm = () => {
  showExpiredDialog.value = false
  // 关闭收银台并回到游戏界面
  emit('close')
  // 触发超时事件，用于日志记录
  setTimeout(() => {
    emit('payment-error', '订单已超时，已返回游戏界面重新下单')
  }, 100)
}

// 处理通用支付失败确认
const handleGenericErrorConfirm = () => {
  showGenericErrorDialog.value = false
  // 重置支付状态，返回表单页面
  resetPaymentStatus()
  currentStep.value = 'form'
  paymentStatus.value = ''
  // 触发错误事件，用于日志记录
  setTimeout(() => {
    emit('payment-error', '通用支付失败，已返回表单页面')
  }, 100)
}

// PayPal 支付相关事件处理
const handlePayPalClose = (reason) => {
  paypalVisible.value = false
  // 如果是从支付失败返回，重置状态并返回表单页面
  if (reason === 'back-to-checkout') {
    resetPaymentStatus()
    currentStep.value = 'form'
    paymentStatus.value = ''
    // 触发错误事件用于日志记录
    setTimeout(() => {
      emit('payment-error', 'PayPal 支付失败，已返回收银台')
    }, 100)
  } else if (reason === 'success-to-game') {
    // 支付成功后点击返回，重定向到游戏界面
    currentPaymentMethod.value = 'paypal'
    orderId.value = 'PP' + Date.now().toString().slice(-10)
    paymentTime.value = formatPaymentTime()
    emit('payment-success', {
      orderId: orderId.value,
      amount: props.productPrice,
      paymentMethod: 'paypal',
      redirectToGame: true
    })
    emit('close')
  }
}

const handlePayPalSuccess = () => {
  paypalVisible.value = false
  // PayPal 支付成功，直接重定向到游戏界面，不显示成功弹窗
  currentPaymentMethod.value = 'paypal'
  orderId.value = 'PP' + Date.now().toString().slice(-10)
  paymentTime.value = formatPaymentTime()
  // 直接触发 payment-success 并重定向
  emit('payment-success', {
    orderId: orderId.value,
    amount: props.productPrice,
    paymentMethod: 'paypal',
    redirectToGame: true
  })
  emit('close')
}

const handlePayPalError = (errorMessage) => {
  // PayPal 支付失败，不在这里处理，让 PayPalPayment 组件显示失败页面
  // 用户需要点击"返回"按钮才能回到收银台表单页面
  // 这里只记录错误日志
  setTimeout(() => {
    emit('payment-error', errorMessage || 'PayPal 支付失败，请稍后重试。')
  }, 100)
}

// “重试”只重置流程状态，保留当前填写/选中的卡信息，方便快速再试一次
const handleRetryPayment = () => {
  paymentStatus.value = ''
  errorMessage.value = ''
  currentStep.value = 'form'
  showExpiredMessage.value = false
  // 保留 cardForm / savedCard 选择，但清除校验错误
  clearErrors()
}

// “返回”作为“重新开始本次支付”，重置到初始状态并清空当前卡信息
const handleReturnToForm = () => {
  resetPaymentStatus()
  // 无论是否有 initialCardData，这里都清空本次填写/选择的卡信息
  cardForm.value = {
    name: '',
    number: '',
    expiry: '',
    cvv: '',
  }
  selectedSavedCardIndex.value = -1
}

// 打开协议
const openTerms = () => {
  window.open('#', '_blank')
}

const openPrivacy = () => {
  window.open('#', '_blank')
}
</script>

<style scoped>
.checkout-overlay {
  position: fixed;
  top: 0;
  right: 0;
  bottom: 0;
  left: 25%;
  background-color: rgba(15, 23, 42, 0.6);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
}

.checkout-modal {
  width: 960px;
  max-width: 96vw;
  max-height: 92vh;
  background-color: #f9fafb;
  border-radius: 12px;
  box-shadow: 0 25px 50px rgba(15, 23, 42, 0.4);
  display: flex;
  flex-direction: column;
  overflow: hidden;
  position: relative;
}

.checkout-modal.browser-window {
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.15), 0 0 0 1px rgba(0, 0, 0, 0.1);
  background-color: #ffffff;
}

/* WebView 顶部栏样式 */
.webview-window {
  background-color: #f3f4f6;
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

.webview-nav-buttons .nav-btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
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

/* 浏览器地址栏 */
.browser-addressbar {
  display: flex;
  align-items: center;
  height: 40px;
  background-color: #f8f8f8;
  border-bottom: 1px solid #e0e0e0;
  padding: 0 8px;
  gap: 8px;
  }

.addressbar-left {
  display: flex;
  gap: 4px;
}

.addressbar-center {
  flex: 1;
  display: flex;
  align-items: center;
}

.addressbar-right {
  display: flex;
  gap: 4px;
}

.nav-btn {
  width: 28px;
  height: 28px;
  border: none;
  background-color: transparent;
  border-radius: 4px;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 16px;
  color: #666;
  transition: background-color 0.2s;
}

.nav-btn:hover {
  background-color: #e0e0e0;
}

.nav-btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
  }

.address-input-wrapper {
  flex: 1;
  display: flex;
  align-items: center;
  background-color: #ffffff;
  border: 1px solid #d0d0d0;
  border-radius: 4px;
  padding: 0 8px;
  height: 28px;
  max-width: 100%;
}

.address-icon {
  font-size: 14px;
  margin-right: 6px;
  color: #666;
}

.address-input {
  flex: 1;
  border: none;
  outline: none;
  font-size: 13px;
  color: #333;
  background: transparent;
  width: 100%;
  }

.address-input:focus {
  outline: none;
}

/* 浏览器内容区域 */
.browser-content {
  flex: 1;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  background-color: #ffffff;
  position: relative;
}

.checkout-content {
  display: flex;
  flex-direction: column;
  height: 100%;
  background-color: #ffffff;
  overflow: hidden;
  min-height: 0;
}

.checkout-header {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  padding: 16px 20px;
}

.checkout-close-btn {
  border: none;
  width: 32px;
  height: 32px;
  border-radius: 6px;
  background-color: transparent;
  color: #6b7280;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  font-size: 20px;
  transition: background-color 0.15s ease, color 0.15s ease;
}

.checkout-close-btn:hover {
  background-color: #f3f4f6;
  color: #111827;
}

.title-area {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.title {
  margin: 0;
  font-size: 20px;
  font-weight: 600;
}

.subtitle {
  margin: 0;
  font-size: 13px;
  color: #9ca3af;
}

.close-button {
  width: 32px;
  height: 32px;
  border-radius: 999px;
  border: none;
  background-color: rgba(15, 23, 42, 0.7);
  color: #e5e7eb;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 20px;
  line-height: 1;
  transition: all 0.15s;
}

.close-button:hover {
  background-color: rgba(15, 23, 42, 1);
}

.checkout-body {
  padding: 24px;
  overflow-y: auto;
  flex: 1;
  min-height: 0;
  position: relative;
}

/* 页面中间的加载spinner */
.page-loading-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1000;
  background-color: rgba(255, 255, 255, 0.8);
}

.page-loading-container {
  width: 80px;
  height: 80px;
  background-color: #374151;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.page-loading-spinner {
  width: 32px;
  height: 32px;
  border: 3px solid rgba(255, 255, 255, 0.3);
  border-top-color: #ffffff;
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}

.checkout-page-header {
  margin-bottom: 16px;
}

.checkout-page-title {
  margin: 0;
  font-size: 18px;
  font-weight: 600;
  color: #111827;
}

.checkout-page-subtitle {
  margin: 4px 0 0;
  font-size: 13px;
  color: #6b7280;
}

.checkout-body.paypal-mode {
  padding: 0;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  height: 100%;
}

.checkout-body.three-ds-mode {
  padding: 0;
  overflow: hidden;
  display: flex;
  flex-direction: column;
  height: 100%;
  align-items: center;
  justify-content: center;
}

.three-ds-container {
  width: 100%;
  height: 100%;
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 40px;
  background-color: #ffffff;
  overflow-y: auto;
}

.three-ds-container .verification-section {
  width: 100%;
  max-width: 520px;
  margin: 0 auto;
}

.checkout-layout {
  display: grid;
  grid-template-columns: 3fr 2fr;
  gap: 24px;
}

.checkout-left,
.checkout-right {
  background-color: #ffffff;
  border-radius: 12px;
  padding: 20px 20px 20px 20px;
  box-shadow: 0 1px 3px rgba(15, 23, 42, 0.06);
}

.payment-methods h3 {
  margin: 0 0 16px;
  font-size: 16px;
  font-weight: 600;
  color: #111827;
}

.method-options {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.method-option {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px 12px;
  border-radius: 8px;
  border: 1.5px solid #e5e7eb;
  background-color: #f9fafb;
  cursor: pointer;
  transition: all 0.15s;
}

.method-option:hover {
  border-color: #d1d5db;
}

.method-option.active {
  border-color: #3b82f6;
  background-color: #eff6ff;
}

.method-icon {
  font-size: 20px;
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.method-name {
  flex: 1;
  font-size: 14px;
  color: #111827;
}

.method-check {
  font-size: 18px;
  color: #3b82f6;
}

.card-brands {
  margin-top: 16px;
  padding-top: 12px;
  border-top: 1px solid #e5e7eb;
}

.brand-label {
  display: block;
  font-size: 12px;
  color: #6b7280;
  margin-bottom: 6px;
}

.brand-icons {
  display: flex;
  gap: 6px;
  flex-wrap: wrap;
}

.brand-icon {
  font-size: 11px;
  padding: 4px 8px;
  border-radius: 4px;
  border: 1px solid #e5e7eb;
  background-color: #ffffff;
  color: #4b5563;
  font-weight: 600;
}

.payment-form {
  margin-top: 20px;
}

.form-group {
  margin-bottom: 14px;
}

.form-group label {
  display: block;
  font-size: 13px;
  font-weight: 500;
  color: #374151;
  margin-bottom: 6px;
}

.form-group input {
  width: 100%;
  padding: 10px 12px;
  border-radius: 6px;
  border: 1.5px solid #d1d5db;
  font-size: 14px;
  color: #111827;
  transition: all 0.15s;
  background-color: #ffffff;
}

.form-group.focused input {
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.15);
}

.form-group.has-error input {
  border-color: #ef4444;
}

.error-message {
  margin-top: 4px;
  font-size: 12px;
  color: #dc2626;
}

.card-number-row {
  display: flex;
  align-items: center;
  gap: 8px;
}

.card-brand-badge {
  padding: 4px 8px;
  border-radius: 999px;
  border: 1px solid #e5e7eb;
  font-size: 11px;
  color: #374151;
  background-color: #f9fafb;
  white-space: nowrap;
}

.form-row {
  display: flex;
  gap: 10px;
}

.form-row .half {
  flex: 1;
}

.save-card-section {
  margin-top: 18px;
  padding-top: 14px;
  border-top: 1px solid #e5e7eb;
}

.save-card-checkbox {
  display: flex;
  align-items: center;
  gap: 10px;
  cursor: pointer;
}

.save-card-checkbox input {
  width: 16px;
  height: 16px;
}

.save-card-text {
  font-size: 13px;
  color: #111827;
  display: flex;
  align-items: center;
  gap: 6px;
}

.save-card-icon {
  color: #10b981;
}

.save-card-desc {
  margin: 6px 0 0 24px;
  font-size: 12px;
  color: #6b7280;
}

.order-summary {
  margin-bottom: 20px;
}

.summary-title {
  margin: 0 0 12px;
  font-size: 15px;
  font-weight: 600;
  color: #111827;
}

.summary-content {
  background-color: #f9fafb;
  border-radius: 8px;
  padding: 12px 12px;
}

.summary-item {
  display: flex;
  justify-content: space-between;
  font-size: 13px;
  color: #4b5563;
  margin-bottom: 6px;
}

.summary-item.total {
  font-weight: 600;
  color: #111827;
}

.summary-label {
  color: #6b7280;
}

.summary-divider {
  height: 1px;
  background-color: #e5e7eb;
  margin: 8px 0;
}

.total-amount {
  font-size: 16px;
}

/* 账单信息样式 */
.billing-info-section {
  margin-top: 20px;
  padding-top: 18px;
  border-top: 1px solid #e5e7eb;
}

.billing-info-title {
  margin: 0 0 14px;
  font-size: 15px;
  font-weight: 600;
  color: #111827;
}

.form-select {
  width: 100%;
  padding: 10px 12px;
  border-radius: 6px;
  border: 1.5px solid #d1d5db;
  font-size: 14px;
  color: #111827;
  background-color: #ffffff;
  cursor: pointer;
  transition: all 0.15s;
}

.form-select:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.15);
}

.save-billing-section {
  margin-top: 14px;
}

.save-billing-checkbox {
  display: flex;
  align-items: center;
  gap: 10px;
  cursor: pointer;
}

.save-billing-checkbox input {
  width: 16px;
  height: 16px;
}

.save-billing-text {
  font-size: 13px;
  color: #111827;
}

/* 右侧账单信息摘要样式 */
.billing-summary {
  margin-top: 20px;
  padding-top: 20px;
  border-top: 1px solid #e5e7eb;
}

.billing-summary-content {
  background-color: #f9fafb;
  border-radius: 8px;
  padding: 16px;
  min-width: 0;
}

.billing-summary-item {
  margin-bottom: 12px;
}

.billing-summary-item:last-child {
  margin-bottom: 0;
}

.billing-label {
  display: block;
  font-size: 13px;
  font-weight: 500;
  color: #374151;
  margin-bottom: 6px;
}

.billing-select {
  width: 100%;
  padding: 8px 10px;
  border-radius: 6px;
  border: 1.5px solid #d1d5db;
  font-size: 13px;
  color: #111827;
  background-color: #ffffff;
  cursor: pointer;
  transition: all 0.15s;
}

.billing-select:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.15);
}

.billing-input {
  width: 100%;
  padding: 8px 10px;
  border-radius: 6px;
  border: 1.5px solid #d1d5db;
  font-size: 13px;
  color: #111827;
  background-color: #ffffff;
  transition: all 0.15s;
}

.billing-input.readonly {
  background-color: #f3f4f6;
  color: #6b7280;
  cursor: not-allowed;
}

.billing-summary-item.has-error .billing-input {
  border-color: #ef4444;
}

.billing-input:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.15);
}

.billing-error-message {
  margin-top: 4px;
  font-size: 11px;
  color: #dc2626;
}

/* 邮箱输入框优化样式 */
.email-input-wrapper {
  margin-bottom: 16px;
}

.email-label {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 8px;
}

.email-hint {
  font-size: 11px;
  font-weight: 400;
  color: #6b7280;
}

.email-input-container {
  position: relative;
  display: flex;
  align-items: center;
  width: 100%;
  min-width: 0;
}

.email-icon {
  position: absolute;
  left: 12px;
  color: #9ca3af;
  pointer-events: none;
  transition: color 0.15s;
  z-index: 1;
}

.email-input-wrapper.is-focused .email-icon {
  color: #3b82f6;
}

.email-input-wrapper.has-error .email-icon {
  color: #ef4444;
}

.email-input {
  width: 100%;
  padding-left: 38px;
  padding-right: 12px;
  padding-top: 10px;
  padding-bottom: 10px;
  font-size: 14px;
  line-height: 1.5;
  box-sizing: border-box;
}

.email-input::placeholder {
  color: #9ca3af;
  font-size: 13px;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.email-input:focus {
  padding-left: 38px;
}

.email-input-container {
  width: 100%;
}

.email-error {
  margin-top: 6px;
  font-size: 12px;
  line-height: 1.4;
  padding-left: 2px;
}

.save-billing-summary {
  margin-top: 12px;
  padding-top: 12px;
  border-top: 1px solid #e5e7eb;
}

.terms-section {
  margin-top: 18px;
}

.terms-checkbox {
  display: flex;
  align-items: flex-start;
  gap: 10px;
}

.terms-checkbox input {
  width: 18px;
  height: 18px;
  margin-top: 2px;
}

.terms-text {
  font-size: 13px;
  color: #374151;
}

.terms-link {
  color: #2563eb;
  text-decoration: underline;
  text-underline-offset: 2px;
}

.checkout-footer {
  margin-top: 20px;
}

.confirm-button {
  width: 100%;
  padding: 12px 18px;
  border-radius: 8px;
  border: none;
  background-color: #2563eb;
  color: #ffffff;
  font-size: 15px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.15s;
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
}

.confirm-button:hover:not(:disabled) {
  background-color: #1d4ed8;
}

.confirm-button:disabled {
  opacity: 0.5;
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

/* 订单超时提示 */
.order-expired-banner {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px 20px;
  background: linear-gradient(90deg, #fef3c7, #fde68a);
  border-bottom: 1px solid #f59e0b;
}

.expired-icon {
  font-size: 20px;
}

.expired-title {
  font-size: 14px;
  font-weight: 600;
  color: #92400e;
}

.expired-message {
  font-size: 13px;
  color: #78350f;
}

/* 支付表单背景模糊效果 */
.checkout-body.blurred {
  filter: blur(2px);
  pointer-events: none;
  user-select: none;
}

/* 支付结果悬浮弹窗遮罩层（覆盖整个收银台窗口） */
.payment-result-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 2000;
  animation: fadeIn 0.2s ease;
}

/* 支付结果弹窗容器 */
.payment-result-modal {
  background-color: #ffffff;
  border-radius: 12px;
  box-shadow: 0 20px 25px -5px rgba(0, 0, 0, 0.1), 0 10px 10px -5px rgba(0, 0, 0, 0.04);
  padding: 40px 32px;
  max-width: 520px;
  width: 90%;
  max-height: 90vh;
  overflow-y: auto;
  animation: slideUp 0.3s ease;
}

.payment-result-content {
  width: 100%;
  text-align: center;
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
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

.spinner-large {
  width: 60px;
  height: 60px;
  border-radius: 999px;
  border: 4px solid #e5e7eb;
  border-top-color: #2563eb;
  margin-bottom: 20px;
  animation: spin 0.8s linear infinite;
}

.result-success,
.result-error,
.result-processing {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 16px;
}

.result-icon {
  width: 72px;
  height: 72px;
  border-radius: 999px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 36px;
  font-weight: 700;
}

.result-icon.success {
  background-color: #dcfce7;
  color: #16a34a;
}

.result-icon.error {
  background-color: #fee2e2;
  color: #ef4444;
}

.order-details {
  width: 100%;
  text-align: left;
  background-color: #f9fafb;
  border-radius: 8px;
  padding: 16px 18px;
}

.detail-item {
  display: flex;
  justify-content: space-between;
  font-size: 13px;
  margin-bottom: 6px;
}

.detail-item:last-child {
  margin-bottom: 0;
}

.error-actions {
  display: flex;
  gap: 10px;
  margin-top: 8px;
}

.result-button {
  flex: 1;
  padding: 10px 16px;
  border-radius: 8px;
  border: none;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.15s;
}

.result-button.primary {
  background-color: #2563eb;
  color: #ffffff;
}

.result-button.primary:hover {
  background-color: #1d4ed8;
}

.result-button.secondary {
  background-color: #ffffff;
  color: #374151;
  border: 1px solid #e5e7eb;
}

.result-button.secondary:hover {
  background-color: #f3f4f6;
}

/* 3DS 样式补充 */
.verification-section {
  max-width: 520px;
  margin: 0 auto;
}

.verification-header {
  margin-bottom: 24px;
}

.verification-icon {
  font-size: 40px;
  margin-bottom: 8px;
}

.verification-desc {
  font-size: 14px;
  color: #6b7280;
}

.verification-methods {
  display: flex;
  flex-direction: column;
  gap: 10px;
  margin-bottom: 20px;
}

.verification-method {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 12px;
  border-radius: 8px;
  border: 1.5px solid #e5e7eb;
  background-color: #ffffff;
  cursor: pointer;
  transition: all 0.15s;
}

.verification-method.active {
  border-color: #2563eb;
  background-color: #eff6ff;
}

.method-icon {
  font-size: 20px;
}

.method-name {
  font-size: 14px;
  font-weight: 500;
}

.method-desc {
  font-size: 12px;
  color: #6b7280;
}

.verification-form .form-group {
  margin-bottom: 14px;
}

.code-input-group {
  display: flex;
  gap: 10px;
}

.code-input {
  flex: 1;
  padding: 10px 12px;
  border-radius: 6px;
  border: 1.5px solid #d1d5db;
  font-size: 14px;
}

.send-code-button {
  padding: 10px 14px;
  border-radius: 6px;
  border: 1px solid #e5e7eb;
  background-color: #f9fafb;
  font-size: 13px;
  cursor: pointer;
}

.send-code-button:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

/* 已保存卡样式 */
.saved-cards-section {
  margin-top: 18px;
  padding-top: 16px;
  border-top: 1px solid #e5e7eb;
}

.saved-cards-title {
  font-size: 13px;
  font-weight: 600;
  color: #111827;
  margin-bottom: 10px;
}

.saved-cards-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.saved-card-option {
  padding: 12px 14px;
  background-color: #f3f4f6;
  border-radius: 10px;
  border: 1.5px solid #e5e7eb;
  transition: all 0.18s ease-out;
  position: relative;
}

.saved-card-option.active {
  border-color: #2563eb;
  box-shadow: 0 0 0 2px rgba(37, 99, 235, 0.6), 0 10px 18px rgba(15, 23, 42, 0.35);
  transform: translateY(-2px) scale(1.01);
}

.saved-card-option:not(.add-new-card-option) {
  cursor: default;
}

.saved-card-row {
  display: flex;
  align-items: center;
  gap: 12px;
  width: 100%;
  cursor: pointer;
}

.saved-card-radio {
  width: 18px;
  height: 18px;
  border-radius: 999px;
  border: 2px solid #cbd5f5;
  background-color: #ffffff;
  position: relative;
  flex-shrink: 0;
}

.saved-card-radio.active {
  border-color: #2563eb;
  box-shadow: 0 0 0 4px rgba(37, 99, 235, 0.15);
}

.saved-card-radio.active::after {
  content: '';
  position: absolute;
  top: 50%;
  left: 50%;
  width: 8px;
  height: 8px;
  border-radius: 999px;
  background: #2563eb;
  transform: translate(-50%, -50%);
}

.saved-card-brand-text {
  font-size: 15px;
  font-weight: 600;
  color: #111827;
  min-width: 60px;
}

.saved-card-mask {
  font-family: 'Menlo', 'Monaco', monospace;
  font-size: 14px;
  color: #111827;
  letter-spacing: 1px;
}

.saved-card-add-text {
  font-size: 14px;
  color: #111827;
}

.saved-card-add-icon {
  margin-left: auto;
  font-size: 18px;
  color: #6b7280;
}

.add-new-card-option {
  background-color: #ffffff;
  border-style: dashed;
}

.add-new-card-option.active {
  background-color: #eff6ff;
  border-color: #93c5fd;
}

.saved-card-delete {
  margin-left: auto;
  width: 28px;
  height: 28px;
  border-radius: 999px;
  border: 1px solid #e5e7eb;
  background: #ffffff;
  color: #ef4444;
  font-size: 14px;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  transition: background-color 0.15s ease, transform 0.1s ease;
}

.saved-card-delete svg {
  width: 16px;
  height: 16px;
}

.saved-card-delete:hover:not(:disabled) {
  background: #fee2e2;
  transform: scale(1.05);
}

.saved-card-delete:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.saved-card-payment-form {
  margin-top: 16px;
}

.saved-card-info-display {
  margin-bottom: 8px;
}

.selected-card-preview {
  background: linear-gradient(135deg, #4f46e5, #7c3aed);
  border-radius: 10px;
  padding: 14px 16px;
  color: #ffffff;
  margin-bottom: 8px;
}

.preview-card-chip {
  width: 32px;
  height: 22px;
  border-radius: 4px;
  background: linear-gradient(135deg, #facc15, #fde047);
  margin-bottom: 8px;
}

.preview-card-number {
  font-family: 'Menlo', 'Monaco', monospace;
  font-size: 14px;
  letter-spacing: 1px;
  margin-bottom: 10px;
}

.preview-card-footer {
  display: flex;
  justify-content: space-between;
  font-size: 11px;
}

.preview-card-brand {
  font-size: 10px;
  margin-top: 6px;
}

.cvv-required-text {
  font-size: 12px;
  color: #6b7280;
  margin: 0;
}

.saved-card-tip {
  margin-top: 8px;
  font-size: 12px;
  color: #6b7280;
}

/* CVV 输入区域：显示在选中卡片下方 */
.saved-card-cvv-section {
  margin-top: 12px;
  padding-top: 12px;
  border-top: 1px solid #e5e7eb;
}

.cvv-label {
  display: block;
  font-size: 13px;
  font-weight: 500;
  color: #374151;
  margin-bottom: 8px;
}

.cvv-input-wrapper {
  position: relative;
  display: flex;
  align-items: center;
  padding: 10px 12px;
  border-radius: 8px;
  border: 1.5px solid #d1d5db;
  background: #ffffff;
  transition: all 0.15s ease;
}

.cvv-input-wrapper:focus-within {
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.15);
}

.cvv-input-wrapper.has-error {
  border-color: #ef4444;
}

.cvv-input-field {
  flex: 1;
  border: none;
  outline: none;
  font-size: 14px;
  color: #111827;
  background: transparent;
  padding-right: 46px;
}

.cvv-card-icon {
  position: absolute;
  right: 10px;
  width: 34px;
  height: 24px;
  display: flex;
  align-items: center;
  justify-content: center;
  pointer-events: none;
}

.cvv-card-icon svg {
  width: 34px;
  height: 24px;
}

.cvv-helper-text {
  margin-top: 6px;
  font-size: 12px;
  color: #6b7280;
}

.cvv-error-message {
  margin-top: 6px;
  font-size: 12px;
  color: #dc2626;
}

/* 取消支付对话框 */
.cancel-dialog-overlay {
  position: fixed;
  inset: 0;
  background-color: rgba(15, 23, 42, 0.55);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 1100;
}

.cancel-dialog {
  width: 360px;
  padding: 20px 22px;
  border-radius: 12px;
  background-color: #ffffff;
  box-shadow: 0 20px 45px rgba(15, 23, 42, 0.4);
}

/* 支付超时弹窗图标 */
.expired-dialog-icon {
  font-size: 48px;
  text-align: center;
  margin-bottom: 16px;
}

/* 通用支付失败弹窗 */
.generic-error-dialog {
  max-width: 420px;
}

.generic-error-icon {
  font-size: 48px;
  text-align: center;
  margin-bottom: 16px;
  color: #ef4444;
}

.generic-error-dialog .cancel-desc {
  margin-bottom: 12px;
  line-height: 1.6;
}

.generic-error-dialog .cancel-desc:last-of-type {
  margin-bottom: 20px;
}

.cancel-title {
  margin: 0 0 8px;
  font-size: 16px;
  font-weight: 600;
  color: #111827;
}

.cancel-desc {
  margin: 0 0 16px;
  font-size: 13px;
  color: #4b5563;
}

.cancel-actions {
  display: flex;
  gap: 10px;
  justify-content: flex-end;
}

.cancel-btn {
  flex: 1;
  padding: 8px 12px;
  border-radius: 8px;
  border: 1px solid transparent;
  font-size: 13px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.15s;
}

.cancel-btn.secondary {
  background-color: #f9fafb;
  border-color: #e5e7eb;
  color: #374151;
}

.cancel-btn.secondary:hover {
  background-color: #e5e7eb;
}

.cancel-btn.primary {
  background-color: #111827;
  color: #ffffff;
}

.cancel-btn.primary:hover {
  background-color: #020617;
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

@media (max-width: 900px) {
  .checkout-modal {
    width: 100%;
    height: 100%;
    max-height: none;
    border-radius: 0;
  }

  .checkout-layout {
    grid-template-columns: 1fr;
  }

  .checkout-overlay {
    left: 0;
  }
}
</style>


