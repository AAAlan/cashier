<template>
  <div v-if="visible" :class="['paypal-overlay', { 'inline-mode': inlineMode, 'webview-mode': !inlineMode }]">
    <div :class="['paypal-container', { 'webview-container': !inlineMode }]">
      <!-- Webview 头部（仅在非 inline 模式显示） -->
      <div v-if="!inlineMode" class="webview-header">
        <div class="webview-header-left">
          <div class="webview-nav-buttons">
            <button class="nav-btn back-btn" title="后退">‹</button>
            <button class="nav-btn forward-btn" title="前进">›</button>
            <button class="nav-btn refresh-btn" title="刷新">↻</button>
          </div>
          <span class="webview-title">PayPal 支付</span>
        </div>
        <div class="webview-header-right">
          <button class="webview-close-btn" @click="handleCancel" title="关闭">✕</button>
        </div>
      </div>

      <!-- PayPal 头部（仅在 inline 模式显示） -->
      <div v-if="inlineMode" class="paypal-header">
        <div class="paypal-logo">
          <div class="logo-text">PayPal</div>
        </div>
        <button class="close-button" @click="handleCancel">×</button>
      </div>

      <!-- PayPal 支付确认页（简化版：仅显示第三方页面提示和跳转按钮） -->
      <div :class="['paypal-content', { 'webview-content': !inlineMode }]">
        <div class="payment-review">
          <div class="third-party-highlight">
            <div class="third-party-badge">跳转三方页面，非自建页面</div>
            <p class="third-party-text">
              您即将跳转至 <strong>PayPal 官方支付页面</strong> 完成本次付款。
            </p>
            <p class="third-party-text">
              本页面仅为演示环境中的模拟界面，<strong>不代表商户自建的实际支付页面</strong>。
            </p>
      </div>

          <div class="third-party-actions">
            <button 
              class="paypal-button primary confirm-button" 
              @click="handleSimulateSuccess"
            >
              在 PayPal 支付成功（模拟）
            </button>
          <button 
              class="paypal-button secondary confirm-button" 
              @click="handleSimulateFailure"
          >
              在 PayPal 支付失败（模拟）
          </button>
          <button 
              class="paypal-button cancel-button" 
              @click="handleCancelToMerchant"
          >
              取消并返回商户
          </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref } from 'vue'

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
  inlineMode: {
    type: Boolean,
    default: false
  }
})

const emit = defineEmits(['close', 'payment-success', 'payment-error'])

const handleCancel = () => {
  // 用户主动关闭 PayPal 页面，返回收银台
  emit('close', 'user-cancelled')
}

const handleCancelToMerchant = () => {
  // 用户点击"取消并返回商户"，关闭 PayPal Webview 并返回收银台，显示取消交易弹窗
  emit('close', 'cancel-to-checkout')
}

const handleSimulateSuccess = () => {
  // 模拟用户在 PayPal 页面支付成功，触发支付成功事件
  // Checkout.vue 的 handlePayPalSuccess 会处理跳转到游戏界面
        emit('payment-success')
}

const handleSimulateFailure = () => {
  // 模拟用户在 PayPal 页面支付失败，返回收银台表单页面
  // 通过 close 事件带上 'back-to-checkout' reason，让 Checkout.vue 处理返回收银台
  emit('close', 'back-to-checkout')
}
</script>

<style scoped>
.paypal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background-color: rgba(0, 0, 0, 0.5);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 10000;
  animation: fadeIn 0.3s ease;
}

.paypal-overlay.inline-mode {
  position: static;
  background-color: transparent;
  z-index: auto;
  animation: none;
  width: 100%;
  height: 100%;
}

.paypal-overlay.webview-mode {
  left: 25%; /* 从调试区右侧开始，只覆盖游戏界面区域 */
  background-color: rgba(0, 0, 0, 0.5);
  align-items: center;
  justify-content: center;
}

.paypal-container {
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

.paypal-overlay.inline-mode .paypal-container {
  width: 100%;
  max-width: 100%;
  max-height: 100%;
  border-radius: 0;
  box-shadow: none;
  animation: none;
}

.paypal-container.webview-container {
  width: 90vw;
  max-width: 1200px;
  height: 90vh;
  border-radius: 12px;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.3);
  animation: slideUp 0.3s ease;
}

.paypal-header {
  padding: 20px 24px;
  border-bottom: 1px solid #e0e0e0;
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: linear-gradient(135deg, #0070ba 0%, #003087 100%);
}

.paypal-logo {
  display: flex;
  align-items: center;
}

.logo-text {
  font-size: 24px;
  font-weight: 700;
  color: #ffffff;
  letter-spacing: 1px;
}

.close-button {
  background: rgba(255, 255, 255, 0.2);
  border: none;
  font-size: 28px;
  color: #ffffff;
  cursor: pointer;
  padding: 0;
  width: 32px;
  height: 32px;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 50%;
  transition: background-color 0.2s;
}

.close-button:hover {
  background: rgba(255, 255, 255, 0.3);
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
  background-color: #0070ba;
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

.step-line {
  flex: 1;
  height: 2px;
  background-color: #e0e0e0;
  margin: 0 12px;
  margin-top: -20px;
}

.step-line.active {
  background-color: #28a745;
}

.paypal-content {
  flex: 1;
  padding: 32px 24px;
  overflow-y: auto;
}

.paypal-content.webview-content {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 40px;
}

/* Webview 头部样式 */
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

.login-section h2,
.payment-review h2 {
  font-size: 24px;
  font-weight: 600;
  color: #333;
  margin-bottom: 24px;
  text-align: center;
}

.login-form {
  max-width: 400px;
  margin: 0 auto;
}

/* 第三方页面高亮提示 */
.third-party-highlight {
  max-width: 480px;
  margin: 0 auto 24px;
  padding: 16px 18px;
  border-radius: 8px;
  border: 1px solid rgba(37, 99, 235, 0.2);
  background: linear-gradient(90deg, #eff6ff, #dbeafe);
}

.third-party-badge {
  display: inline-flex;
  align-items: center;
  padding: 2px 8px;
  border-radius: 999px;
  font-size: 11px;
  font-weight: 600;
  color: #1d4ed8;
  background-color: #e0ecff;
  margin-bottom: 8px;
}

.third-party-text {
  margin: 0 0 4px;
  font-size: 13px;
  color: #1f2937;
  line-height: 1.6;
}

.third-party-text:last-of-type {
  margin-bottom: 0;
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
  border-color: #0070ba;
}

.form-group input:disabled {
  background-color: #f5f5f5;
  cursor: not-allowed;
}

.paypal-button {
  width: 100%;
  padding: 14px 24px;
  border: none;
  border-radius: 6px;
  font-size: 16px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.paypal-button.primary {
  background-color: #0070ba;
  color: #ffffff;
}

.paypal-button.primary:hover:not(:disabled) {
  background-color: #005ea6;
}

.paypal-button.secondary {
  background-color: #f5f5f5;
  color: #333;
  border: 1px solid #e0e0e0;
}

.paypal-button.secondary:hover:not(:disabled) {
  background-color: #e0e0e0;
}

.paypal-button.cancel-button {
  background-color: #ffffff;
  color: #6b7280;
  border: 1px solid #d1d5db;
  margin-top: 12px;
}

.paypal-button.cancel-button:hover:not(:disabled) {
  background-color: #f9fafb;
  color: #374151;
  border-color: #9ca3af;
}

.paypal-button:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.login-help {
  text-align: center;
  margin-top: 16px;
}

.help-link {
  color: #0070ba;
  text-decoration: none;
  font-size: 14px;
}

.help-link:hover {
  text-decoration: underline;
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

.payment-method-select {
  margin-bottom: 24px;
}

.payment-method-select h3 {
  font-size: 16px;
  font-weight: 600;
  color: #333;
  margin-bottom: 12px;
}

.method-options {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.method-option {
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

.method-option:hover {
  border-color: #0070ba;
}

.method-option.active {
  border-color: #0070ba;
  background-color: #f0f7ff;
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

.confirm-button {
  margin-top: 8px;
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
  border-top-color: #0070ba;
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

.error-actions .paypal-button {
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

