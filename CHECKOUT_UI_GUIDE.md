# 收银台UI设计指南

## 设计理念

收银台UI设计参考了 **Adyen** 和 **Stripe** 的设计风格，采用简洁、专业、现代的设计理念，提供流畅的支付体验。

## 设计特点

### 1. 简洁专业
- 清晰的视觉层次
- 专业的表单设计
- 统一的间距和字体
- 简洁的颜色系统

### 2. 实时验证
- 输入时实时格式化
- 即时错误提示
- 卡片品牌自动识别
- Luhn算法验证

### 3. 安全信任
- 安全标识显示
- 加密提示
- 清晰的支付流程
- 专业的UI设计

### 4. 响应式设计
- 移动端优化
- 自适应布局
- 触摸友好
- 键盘适配

## 组件说明

### AdyenStyleCheckout.vue

主收银台组件，包含：
- 订单摘要
- 支付方式选择
- 银行卡支付表单
- 实时验证和格式化

**主要特性：**
- 支付方式单选（Radio风格）
- 卡片号自动格式化（4位一组）
- 卡片品牌自动识别（Visa, Mastercard, Amex等）
- 实时表单验证
- CVV提示工具
- 安全标识

**使用示例：**

```vue
<template>
  <AdyenStyleCheckout
    :visible="showCheckout"
    :product-name="'Game Item'"
    :product-price="9.99"
    :default-currency="'USD'"
    :supported-payment-methods="['card', 'paypal', 'alipay']"
    @close="showCheckout = false"
    @jump-to-card="handleCardPayment"
    @jump-to-paypal="handlePayPalPayment"
    @payment-success="handleSuccess"
  />
</template>
```

### AdyenStyleCardPayment.vue

银行卡支付组件，包含：
- 卡片信息预览
- 3D Secure验证
- 支付处理状态
- 支付结果展示

**主要特性：**
- 精美的卡片预览
- 多种验证方式（SMS、Email）
- 验证码自动填入（演示）
- 倒计时重发
- 支付状态动画
- 错误处理

**使用示例：**

```vue
<template>
  <AdyenStyleCardPayment
    :visible="showCardPayment"
    :product-name="'Game Item'"
    :product-price="9.99"
    :card-form="cardForm"
    :currency="'USD'"
    :payment-scenario="'normal'"
    @close="showCardPayment = false"
    @payment-success="handleSuccess"
    @payment-error="handleError"
  />
</template>
```

## 设计规范

### 颜色系统

```css
/* 主色调 */
--primary: #111827;      /* 深灰黑 */
--secondary: #6b7280;    /* 中灰 */
--border: #e5e7eb;       /* 浅灰边框 */
--bg: #f9fafb;           /* 浅灰背景 */

/* 状态色 */
--success: #10b981;      /* 绿色 */
--error: #ef4444;        /* 红色 */
--warning: #f59e0b;      /* 橙色 */
--info: #3b82f6;         /* 蓝色 */
```

### 字体系统

```css
/* 标题 */
font-size: 20px;
font-weight: 600;
color: #111827;

/* 副标题 */
font-size: 14px;
color: #6b7280;

/* 正文 */
font-size: 15px;
color: #111827;

/* 辅助文本 */
font-size: 13px;
color: #6b7280;
```

### 间距系统

```css
/* 小间距 */
padding: 8px, 12px;

/* 中间距 */
padding: 16px, 20px;

/* 大间距 */
padding: 24px, 32px;
```

### 圆角系统

```css
/* 小圆角 */
border-radius: 6px;

/* 中圆角 */
border-radius: 8px;

/* 大圆角 */
border-radius: 12px;
```

## 交互设计

### 表单输入

1. **焦点状态**
   - 边框颜色变化：`#d1d5db` → `#111827`
   - 添加阴影：`box-shadow: 0 0 0 3px rgba(17, 24, 39, 0.1)`

2. **错误状态**
   - 边框颜色：`#ef4444`
   - 显示错误信息
   - 实时验证

3. **禁用状态**
   - 透明度：`opacity: 0.6`
   - 鼠标样式：`cursor: not-allowed`

### 按钮交互

1. **悬停效果**
   - 背景色加深
   - 平滑过渡：`transition: all 0.15s`

2. **点击反馈**
   - 轻微缩放
   - 状态变化

3. **加载状态**
   - 显示加载动画
   - 禁用交互

### 动画效果

```css
/* 淡入 */
@keyframes fadeIn {
  from { opacity: 0; }
  to { opacity: 1; }
}

/* 上滑 */
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

/* 旋转 */
@keyframes spin {
  to { transform: rotate(360deg); }
}
```

## 最佳实践

### 1. 表单验证

- **实时验证**：用户输入时立即验证
- **清晰错误**：明确的错误提示
- **友好提示**：帮助用户理解问题

### 2. 支付流程

- **步骤清晰**：明确的步骤指示
- **状态反馈**：实时显示处理状态
- **错误处理**：友好的错误提示和重试

### 3. 安全提示

- **安全标识**：显示安全锁图标
- **加密提示**：说明数据加密
- **信任元素**：专业的UI设计

### 4. 移动端优化

- **触摸友好**：按钮大小适中
- **键盘适配**：正确的输入类型
- **响应式布局**：自适应屏幕

## 与Adyen/Stripe的对比

### 相似之处

1. ✅ 简洁的白色背景
2. ✅ 清晰的表单字段
3. ✅ 实时输入验证
4. ✅ 专业的卡片输入
5. ✅ 流畅的动画
6. ✅ 清晰的错误提示
7. ✅ 安全标识

### 我们的特色

1. 🎨 支持多种支付方式
2. 🌍 多币种支持
3. 🔒 3D Secure验证
4. 📱 移动端优化
5. 🎯 游戏场景适配

## 使用建议

1. **保持一致性**：使用统一的组件和样式
2. **测试验证**：确保表单验证正确
3. **优化性能**：减少不必要的重渲染
4. **错误处理**：友好的错误提示
5. **用户体验**：流畅的交互反馈

## 更新日志

### v1.0.0 (2024-01-15)
- ✅ 创建Adyen/Stripe风格的收银台组件
- ✅ 实现实时表单验证
- ✅ 添加卡片品牌识别
- ✅ 优化移动端体验
- ✅ 添加安全标识

