<script setup lang="ts">
import { ref, computed, watch, onMounted, onUnmounted } from 'vue';

// 32位二进制数据
const binaryValue = ref(new Array(32).fill(0));
const inputValue = ref('0');
const isValidExpression = ref(true);

// 计算属性 - 不同进制的显示结果
const valueInDifferentFormats = computed(() => {
  // 将二进制数组转换为整数值
  const binaryString = binaryValue.value.join('');
  // 使用 >>> 0 确保我们得到的是无符号32位整数的数值表示
  const unsignedDecimalValue = (parseInt(binaryString, 2) || 0) >>> 0;

  // 计算有符号整数值 (int32_t)
  // 使用 | 0 将无符号32位整数的位模式解释为有符号32位整数
  const signedValue = unsignedDecimalValue | 0;

  return {
    // 使用无符号值进行十六进制转换
    Hex: '0x' + unsignedDecimalValue.toString(16).toUpperCase(),
    // 显示无符号十进制值
    Dec: unsignedDecimalValue.toString(), // 无符号十进制 (uDec)
    // 显示有符号十进制值
    SignedDec: signedValue.toString(), // 有符号十进制 (Dec)
  };
});

// 切换单个位的值 (0->1, 1->0)
function toggleBit(index: number) {
  const newBinaryValue = [...binaryValue.value];
  newBinaryValue[index] = newBinaryValue[index] === 0 ? 1 : 0;
  binaryValue.value = newBinaryValue;
  
  // 更新输入框的值以匹配二进制值
  // 默认使用十六进制表示
  const binaryString = newBinaryValue.join('');
  const decimalValue = parseInt(binaryString, 2) || 0;
  inputValue.value = '0x' + decimalValue.toString(16).toUpperCase();
}

// 验证表达式是否合法
function validateExpression(expr: string): boolean {
  try {
    // 处理十六进制 (0x) 前缀
    const hexRegex = /0x[0-9a-f]+/gi;
    let cleanExpr = expr.replace(hexRegex, match => {
      return parseInt(match, 16).toString();
    });

    // 处理二进制 (0b) 前缀
    const binRegex = /0b[01]+/gi;
    cleanExpr = cleanExpr.replace(binRegex, match => {
      return parseInt(match.substring(2), 2).toString();
    });

    // 处理八进制 (0) 前缀
    const octRegex = /\b0[0-7]+\b/g;
    cleanExpr = cleanExpr.replace(octRegex, match => {
      return parseInt(match, 8).toString();
    });

    // 尝试评估表达式
    // eslint-disable-next-line no-new-func
    Function(`"use strict"; return (${cleanExpr})`)();
    return true;
  } catch (e) {
    return false;
  }
}

// 计算表达式的值
function evaluateExpression(expr: string): number {
  try {
    // 处理十六进制 (0x) 前缀
    const hexRegex = /0x[0-9a-f]+/gi;
    let cleanExpr = expr.replace(hexRegex, match => {
      return parseInt(match, 16).toString();
    });

    // 处理二进制 (0b) 前缈
    const binRegex = /0b[01]+/gi;
    cleanExpr = cleanExpr.replace(binRegex, match => {
      return parseInt(match.substring(2), 2).toString();
    });

    // 处理八进制 (0) 前缀
    const octRegex = /\b0[0-7]+\b/g;
    cleanExpr = cleanExpr.replace(octRegex, match => {
      return parseInt(match, 8).toString();
    });

    // 使用Function构造函数评估表达式
    // eslint-disable-next-line no-new-func
    const result = Function(`"use strict"; return (${cleanExpr})`)();
    
    // 确保结果是一个有效的数字且在32位范围内
    const numberResult = Number(result);
    if (isNaN(numberResult)) return 0;
    
    // 将结果限制在32位整数范围内
    return numberResult >>> 0; // 无符号右移0位将数字转换为32位无符号整数
  } catch (e) {
    console.error('Expression evaluation error:', e);
    return 0;
  }
}

// 从输入框更新二进制值
function updateFromInput() {
  try {
    const inputStr = inputValue.value.trim();
    if (!inputStr) return;
    
    // 验证表达式合法性
    isValidExpression.value = validateExpression(inputStr);
    if (!isValidExpression.value) return;
    
    // 计算表达式的值
    const decimalValue = evaluateExpression(inputStr);
    
    // 转换为二进制数组
    const binaryStr = decimalValue.toString(2).padStart(32, '0');
    binaryValue.value = binaryStr.split('').map(bit => parseInt(bit, 10));
  } catch (e) {
    console.error('Invalid input:', e);
    isValidExpression.value = false;
  }
}

// 构造4行8位的数据结构
const bitGroups = computed(() => {
  const groups = [];
  for (let row = 0; row < 4; row++) {
    const rowBits = [];
    for (let col = 0; col < 8; col++) {
      const index = row * 8 + col;
      const bitPosition = 31 - index; // 位编号从31到0
      rowBits.push({
        index: index,
        position: bitPosition,
        value: binaryValue.value[index]
      });
    }
    groups.push(rowBits);
  }
  return groups;
});

// 监听输入值变化，实时验证表达式
watch(inputValue, (newValue) => {
  if (newValue.trim() !== '') {
    // 仅验证表达式的合法性，但不立即计算
    isValidExpression.value = validateExpression(newValue);
  } else {
    isValidExpression.value = true;
  }
}, { immediate: true });

// Debounce mechanism for input updates
let debounceTimer: number | undefined;

// 监听输入值变化，延迟更新二进制位显示 (using debounce)
watch(inputValue, (newValue) => {
  clearTimeout(debounceTimer); // Clear previous timer
  debounceTimer = setTimeout(() => {
    if (newValue.trim() !== '' && isValidExpression.value) {
      updateFromInput();
    }
  }, 300); // 300ms debounce delay
});

// 添加响应式宽度变量
const bitButtonWidth = ref(2); // 默认位按钮宽度(rem)

// 计算位组间隔
const bitGroupGap = computed(() => {
  return bitButtonWidth.value * 0.25 + 'rem';
});

// 监听窗口大小变化
function updateBitButtonWidth() {
  const windowWidth = window.innerWidth;
  // 计算适合的按钮宽度 (rem)
  if (windowWidth < 500) {
    bitButtonWidth.value = 1.5;
  } else if (windowWidth < 768) {
    bitButtonWidth.value = 1.8;
  } else {
    bitButtonWidth.value = 2;
  }
}

// 组件挂载和窗口大小变化时更新按钮宽度
onMounted(() => {
  updateBitButtonWidth();
  window.addEventListener('resize', updateBitButtonWidth);
});

// 组件卸载时移除事件监听
onUnmounted(() => {
  window.removeEventListener('resize', updateBitButtonWidth);
});
</script>

<template>
  <div class="flex flex-col min-h-screen p-4 font-sans md-app-container md-dark-mode no-scrollbar">
    <!-- 位显示区域 - Material Design 风格 -->
    <div class="w-full mb-6 md-card">
      <div class="flex justify-center" :style="{ gap: bitGroupGap }">
        <!-- 左侧2行 (31-16) -->
        <div class="flex flex-col" :style="{ gap: bitGroupGap }">
          <!-- 31-24 位 -->
          <div class="bit-group-container">
            <div v-for="(row, rowIndex) in [bitGroups[0]]" :key="`row-left-top-${rowIndex}`">
              <!-- 位编号显示 -->
              <div class="flex">
                <div v-for="bit in row" :key="`header-${bit.position}`" 
                     class="md-bit-header">
                  {{ bit.position }}
                </div>
              </div>
              <!-- 位值显示 -->
              <div class="flex">
                <div v-for="bit in row" :key="`bit-${bit.index}`"
                     class="md-bit-value"
                     :class="[bit.value === 1 ? 'md-bit-on' : 'md-bit-off']"
                     @click="toggleBit(bit.index)">
                  {{ bit.value }}
                </div>
              </div>
            </div>
          </div>
          
          <!-- 23-16 位 -->
          <div class="bit-group-container">
            <div v-for="(row, rowIndex) in [bitGroups[1]]" :key="`row-left-bottom-${rowIndex}`">
              <!-- 位编号显示 -->
              <div class="flex">
                <div v-for="bit in row" :key="`header-${bit.position}`" 
                     class="md-bit-header">
                  {{ bit.position }}
                </div>
              </div>
              <!-- 位值显示 -->
              <div class="flex">
                <div v-for="bit in row" :key="`bit-${bit.index}`"
                     class="md-bit-value"
                     :class="[bit.value === 1 ? 'md-bit-on' : 'md-bit-off']"
                     @click="toggleBit(bit.index)">
                  {{ bit.value }}
                </div>
              </div>
            </div>
          </div>
        </div>
        
        <!-- 右侧2行 (15-0) -->
        <div class="flex flex-col" :style="{ gap: bitGroupGap }">
          <!-- 15-8 位 -->
          <div class="bit-group-container">
            <div v-for="(row, rowIndex) in [bitGroups[2]]" :key="`row-right-top-${rowIndex}`">
              <!-- 位编号显示 -->
              <div class="flex">
                <div v-for="bit in row" :key="`header-${bit.position}`" 
                     class="md-bit-header">
                  {{ bit.position }}
                </div>
              </div>
              <!-- 位值显示 -->
              <div class="flex">
                <div v-for="bit in row" :key="`bit-${bit.index}`"
                     class="md-bit-value"
                     :class="[bit.value === 1 ? 'md-bit-on' : 'md-bit-off']"
                     @click="toggleBit(bit.index)">
                  {{ bit.value }}
                </div>
              </div>
            </div>
          </div>
          
          <!-- 7-0 位 -->
          <div class="bit-group-container">
            <div v-for="(row, rowIndex) in [bitGroups[3]]" :key="`row-right-bottom-${rowIndex}`">
              <!-- 位编号显示 -->
              <div class="flex">
                <div v-for="bit in row" :key="`header-${bit.position}`" 
                     class="md-bit-header">
                  {{ bit.position }}
                </div>
              </div>
              <!-- 位值显示 -->
              <div class="flex">
                <div v-for="bit in row" :key="`bit-${bit.index}`"
                     class="md-bit-value"
                     :class="[bit.value === 1 ? 'md-bit-on' : 'md-bit-off']"
                     @click="toggleBit(bit.index)">
                  {{ bit.value }}
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
    
    <!-- Material Design 分隔线 -->
    <div class="md-divider"></div>
    
    <!-- 计算器输入区域 - 放在位显示和进制显示之间 -->
    <div class="w-full mb-6 md-card">
      <div class="calc-input-wrapper">
        <div class="calc-label-container" :class="isValidExpression ? 'calc-valid' : 'calc-invalid'">
          <span class="calc-label">Calc</span>
        </div>
        <div class="md-input-container overflow-hidden">
          <input 
            type="text" 
            v-model="inputValue" 
            @change="updateFromInput" 
            @keydown.enter="updateFromInput" 
            class="md-input"
            placeholder="输入: 0x1F + 5 * (3 - 1)"
          />
          <div class="md-input-underline"></div>
        </div>
      </div>
    </div>
    
    <!-- Material Design 分隔线 -->
    <div class="md-divider"></div>
    
    <!-- 进制显示区域 - Material Design 风格 -->
    <div class="w-full">
      <div class="md-card">
        <!-- 格式显示区域 - 改进的矩形包裹样式 -->
        <div class="md-results md-format-container">
          <!-- 进制值包装 - 带有矩形外框和内部圆角矩形 -->
          <div class="md-format-box">
            <div class="md-format-label">Hex</div>
            <div class="md-format-value-outer">
              <div class="md-format-value-inner">
                {{ valueInDifferentFormats.Hex }}
              </div>
            </div>
          </div>
          
          <div class="md-format-box">
            <div class="md-format-label">uDec</div>
            <div class="md-format-value-outer">
              <div class="md-format-value-inner">
                {{ valueInDifferentFormats.Dec }}
              </div>
            </div>
          </div>
          
          <div class="md-format-box">
            <div class="md-format-label">Dec</div>
            <div class="md-format-value-outer">
              <div class="md-format-value-inner">
                {{ valueInDifferentFormats.SignedDec }}
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style>
:root {
  /* Material Design Color System */
  --md-primary: #6200ee;  /* Deep purple 500 */
  --md-primary-variant: #3700b3;
  --md-secondary: #03dac6;  /* Teal 200 */
  --md-secondary-variant: #018786;
  --md-background: #ffffff;
  --md-surface: #ffffff;
  --md-error: #b00020;
  --md-on-primary: #ffffff;
  --md-on-secondary: #000000;
  --md-on-background: #000000;
  --md-on-surface: #000000;
  --md-on-error: #ffffff;
  
  /* Typography */
  --md-font-family: Roboto, 'Segoe UI', system-ui, -apple-system, sans-serif;
  
  /* Elevation Shadows */
  --md-elevation-1: 0 2px 1px -1px rgba(0,0,0,0.2), 0 1px 1px 0 rgba(0,0,0,0.14), 0 1px 3px 0 rgba(0,0,0,0.12);
  --md-elevation-2: 0 3px 1px -2px rgba(0,0,0,0.2), 0 2px 2px 0 rgba(0,0,0,0.14), 0 1px 5px 0 rgba(0,0,0,0.12);
  --md-elevation-3: 0 3px 3px -2px rgba(0,0,0,0.2), 0 3px 4px 0 rgba(0,0,0,0.14), 0 1px 8px 0 rgba(0,0,0,0.12);
  --md-elevation-4: 0 2px 4px -1px rgba(0,0,0,0.2), 0 4px 5px 0 rgba(0,0,0,0.14), 0 1px 10px 0 rgba(0,0,0,0.12);
  --md-elevation-8: 0 5px 5px -3px rgba(0,0,0,0.2), 0 8px 10px 1px rgba(0,0,0,0.14), 0 3px 14px 2px rgba(0,0,0,0.12);
}

/* Base Application Styles */
body {
  font-family: var(--md-font-family);
  margin: 0;
  padding: 0;
  background-color: var(--md-background);
  color: var(--md-on-background);
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

/* Removing page container and app container styles */

.md-app-container {
  background-color: var(--md-background);
  min-height: 100vh;
  transition: background-color 0.3s;
}

/* Material Design Card */
.md-card {
  background-color: var(--md-surface);
  border-radius: 8px;
  padding: 16px;
  box-shadow: var(--md-elevation-2);
  transition: box-shadow 0.3s, background-color 0.3s;
  margin: 0 4px; /* Reduced horizontal margin */
  width: calc(100% - 8px); /* Ensure width calculation includes reduced margin */
  box-sizing: border-box;
}

/* Typography */
.md-title {
  font-size: 1.25rem;
  font-weight: 500;
  letter-spacing: 0.0125em;
  color: var(--md-on-surface);
  margin-bottom: 1rem;
  text-align: center;
  transition: color 0.3s;
}

.md-title-inline {
  font-size: 1.25rem;
  font-weight: 500;
  letter-spacing: 0.0125em;
  color: var(--md-on-surface);
  margin-bottom: 0;
  white-space: nowrap;
  transition: color 0.3s;
}

/* Bit Display */
.md-bit-header {
  width: v-bind('bitButtonWidth + "rem"');
  height: v-bind('bitButtonWidth * 0.75 + "rem"');
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: v-bind('bitButtonWidth * 0.375 + "rem"');
  font-weight: 500;
  background-color: var(--md-surface);
  color: var(--md-on-surface);
  opacity: 0.7;
  border-radius: 0;
  transition: background-color 0.3s, color 0.3s;
}

.md-bit-value {
  width: v-bind('bitButtonWidth + "rem"');
  height: v-bind('bitButtonWidth + "rem"');
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: 4px;
  cursor: pointer;
  font-weight: 500;
  transition: all 0.2s ease-in-out;
  user-select: none;
  box-shadow: var(--md-elevation-1);
}

.md-bit-on {
  background-color: var(--md-primary);
  color: var(--md-on-primary);
  box-shadow: var(--md-elevation-2);
  transform: translateY(-1px);
}

.md-bit-off {
  background-color: var (--md-surface);
  color: var(--md-on-surface);
}

.md-bit-off:hover {
  background-color: rgba(98, 0, 238, 0.08);  /* Primary color with 8% opacity */
  box-shadow: var(--md-elevation-2);
}

.md-bit-on:active, .md-bit-off:active {
  transform: translateY(0);
  box-shadow: var(--md-elevation-1);
}

/* 位组容器样式 */
.bit-group-container {
  padding: 8px;
  border-radius: 4px;
  background-color: rgba(255, 255, 255, 0.05);
  border: 1px solid rgba(255, 255, 255, 0.1);
}

.md-dark-mode .bit-group-container {
  background-color: rgba(255, 255, 255, 0.03);
  border: 1px solid rgba(255, 255, 255, 0.07);
}

/* Calc input container - new responsive layout */
.calc-input-wrapper {
  display: flex;
  flex-direction: column;
  width: 100%;
  padding: 4px 0; /* Keep vertical padding, remove horizontal */
  box-sizing: border-box;
}

@media (min-width: 640px) {
  .calc-input-wrapper {
    flex-direction: row;
    align-items: center;
  }
}

/* 计算器标签样式 - Reduced size */
.calc-label-container {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 6px 10px; /* Reduced padding */
  border: 1px solid var(--md-secondary);
  border-radius: 4px;
  height: 36px; /* Reduced height */
  min-width: 60px; /* Reduced min-width */
  width: fit-content;
  max-width: calc(100% - 16px); /* Adjust max-width for new margins */
  transition: all 0.3s ease;
  text-align: center;
  margin: 0 8px 10px 8px; /* Adjusted margins */
  background-color: rgba(3, 218, 198, 0.05);
}

@media (min-width: 640px) {
  .calc-label-container {
    margin: 0 12px 0 8px; /* Adjusted right margin */
    flex-shrink: 0;
  }
}

/* Input Container Styling */
.md-input-container {
  position: relative;
  display: flex;
  align-items: center;
  width: 100%;
  margin: 0 8px; /* Adjusted margin */
  min-width: 0;
  box-sizing: border-box;
  max-width: calc(100% - 16px); /* Ensure container doesn't overflow */
}

/* 修改输入样式，移除颜色变化 */
.md-input {
  width: 100%;
  padding: 10px 0; /* Reduced vertical padding */
  font-size: 16px;
  font-family: 'Roboto Mono', monospace;
  background: transparent;
  border: none;
  border-bottom: 1px solid rgba(255, 255, 255, 0.38);
  border-radius: 4px 4px 0 0;
  outline: none;
  transition: all 0.3s;
  color: var(--md-on-surface);
  box-sizing: border-box;
  margin: 0;
  text-overflow: ellipsis;
  overflow: hidden;
  white-space: nowrap;
}

.md-input:focus {
  background-color: rgba(255, 255, 255, 0.05);
}

.md-input-underline {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  height: 2px;
  background: var(--md-primary);
  transform-origin: center;
  transform: scaleX(0);
  transition: transform 0.3s;
}

.md-input:focus ~ .md-input-underline {
  transform: scaleX(1);
}

/* Results Section */
.md-results {
  padding-top: 8px;
}

.md-format-label {
  text-align: center;
  font-size: 0.875rem;
  font-weight: 500;
  letter-spacing: 0.0178571429em;
  color: var(--md-on-surface);
  opacity: 0.7;
  transition: color 0.3s;
}

.md-format-value {
  background-color: rgba(0, 0, 0, 0.04);
  border-radius: 4px;
  padding: 12px 8px;
  text-align: center;
  font-family: 'Roboto Mono', monospace;
  font-size: 0.875rem;
  color: var (--md-on-surface);
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  transition: background-color 0.3s, color 0.3s;
}

/* 新的进制格式显示样式 */
.md-format-container {
  display: flex;
  justify-content: space-between;
  gap: 12px;
}

.md-format-box {
  flex: 1;
  border: 1px solid rgba(255, 255, 255, 0.12);
  border-radius: 4px;
  padding: 8px;
  display: flex;
  flex-direction: column;
}

.md-format-label {
  text-align: center;
  font-size: 0.875rem;
  font-weight: 500;
  letter-spacing: 0.0178571429em;
  color: var(--md-on-surface);
  opacity: 0.7;
  margin-bottom: 8px;
  transition: color 0.3s;
}

/* 外部矩形 */
.md-format-value-outer {
  padding: 2px;
  border: 1px solid rgba(255, 255, 255, 0.1);
}

/* 内部圆角矩形 */
.md-format-value-inner {
  background-color: rgba(255, 255, 255, 0.08);
  border-radius: 4px;
  padding: 10px 8px;
  text-align: center;
  font-family: 'Roboto Mono', monospace;
  font-size: 0.875rem;
  color: var(--md-on-surface);
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
  transition: background-color 0.3s, color 0.3s;
}

/* 模块分隔线样式 */
.md-divider {
  height: 1px;
  background: rgba(255, 255, 255, 0.12); /* Simple gray divider */
  margin: 12px 0;
}

/* 强制使用暗黑模式 */
.md-dark-mode {
  --md-primary: #bb86fc;  /* Light purple for dark theme */
  --md-primary-variant: #3700b3;
  --md-secondary: #03dac6;
  --md-secondary-variant: #03dac6;
  --md-background: #121212;
  --md-surface: #121212;
  --md-error: #cf6679;
  --md-on-primary: #000000;
  --md-on-secondary: #000000;
  --md-on-background: #ffffff;
  --md-on-surface: #ffffff;
  --md-on-error: #000000;
  
  /* Dark Mode Elevation Overlays */
  --md-surface-overlay-1: rgba(255, 255, 255, 0.05);
  --md-surface-overlay-2: rgba(255, 255, 255, 0.08);
  --md-surface-overlay-3: rgba(255, 255, 255, 0.11);
  
  background-color: var(--md-background);
}

.md-dark-mode .md-card {
  background-color: var(--md-surface);
  box-shadow: var(--md-elevation-1);
  background-image: linear-gradient(var(--md-surface-overlay-1), var(--md-surface-overlay-1));
}

.md-dark-mode .md-bit-header {
  background-color: transparent;
  color: var(--md-on-surface);
}

.md-dark-mode .md-bit-off {
  background-color: rgba(255, 255, 255, 0.05);
  color: var(--md-on-surface);
}

.md-dark-mode .md-bit-on {
  background-color: var(--md-primary);
  color: var(--md-on-primary);
}

.md-dark-mode .md-bit-off:hover {
  background-color: rgba(187, 134, 252, 0.12);  /* Primary color with 12% opacity */
}

.md-dark-mode .md-input {
  border-bottom-color: rgba(255, 255, 255, 0.38);
  color: var(--md-on-surface);
}

.md-dark-mode .md-format-value {
  background-color: rgba(255, 255, 255, 0.08);  /* Surface overlay 2 */
}

/* 计算器标签样式 */
.calc-label-container {
  display: flex;
  align-items: center;
  justify-content: center;
  padding: 6px 10px; /* Reduced padding */
  border: 1px solid var(--md-secondary);
  border-radius: 4px;
  height: 36px; /* Reduced height */
  min-width: 60px; /* Reduced min-width */
  max-width: 100%;
  transition: all 0.3s ease;
  text-align: center;
}

.calc-valid {
  border-color: var(--md-secondary-variant);
  color: var(--md-secondary-variant);
  background-color: rgba(3, 218, 198, 0.08);
}

.calc-invalid {
  border-color: var(--md-error);
  color: var(--md-error);
  background-color: rgba(207, 102, 121, 0.08);
}

.md-dark-mode .calc-valid {
  border-color: var(--md-secondary);
  color: var(--md-secondary);
}

.md-dark-mode .calc-invalid {
  border-color: var(--md-error);
  color: var(--md-error);
}

.calc-label {
  font-size: 1.1rem; /* Reduced font size */
  font-weight: 500;
  letter-spacing: 0.0125em;
  margin: 0;
  white-space: nowrap;
  transition: color 0.3s;
  display: block;
  width: 100%;
  color: var(--md-secondary); /* Match screenshot teal color */
}

/* Media Queries for Responsive Layout */
@media (max-width: 640px) {
  .calc-label-container {
    width: calc(100% - 16px); /* Keep margin on small screens */
    margin-left: 8px;
    margin-right: 8px;
  }
  
  .md-input-container {
    width: calc(100% - 16px); /* Keep margin on small screens */
    margin-left: 8px;
    margin-right: 8px;
  }
  
  .md-card {
    padding: 16px 8px; /* Adjust card padding on small screens */
    margin: 0 4px; /* Keep reduced horizontal margins */
  }
}

/* Hide scrollbars and prevent overflow */
.no-scrollbar {
  -ms-overflow-style: none;  /* IE and Edge */
  scrollbar-width: none;     /* Firefox */
  overflow: hidden;          /* Hide overflow */
}

.no-scrollbar::-webkit-scrollbar {
  display: none;  /* Chrome, Safari and Opera */
}
</style>