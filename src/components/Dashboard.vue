<template>
  <div class="app-container">
    <div class="dashboard-card">
      <div class="header-section">
        <div class="base-circle"></div>
        <h2 class="title-text">Base 资产看板</h2>
      </div>
      
      <div class="status-box">
        <p v-if="!account" class="text-warning">⚠️ 请先连接 Base 钱包</p>
        <p v-else-if="chainId !== 8453n" class="text-danger">❌ 当前不在 Base 网络</p>
        <p v-else class="text-success">✅ 已成功接入 Base 生态</p>
        <p v-if="account" class="address-tag">{{ shortAddress }}</p>
      </div>

      <button 
        @click="connectBase" 
        :class="['btn-base', { 'btn-warning': account && chainId !== 8453n }]"
      >
        {{ buttonText }}
      </button>

      <div v-show="account && chainId === 8453n" class="chart-section">
        <div ref="chartRef" class="echarts-dom"></div>
        <div class="balance-display">
          <span class="label">当前余额:</span>
          <span class="value">{{ balance }} ETH</span>
        </div>
      </div>

      <div class="footer">
        <p>Built by WindyBuilds @ Base Ecosystem</p>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed, watch } from 'vue';
import { ethers } from 'ethers';
import * as echarts from 'echarts';

// --- 响应式状态 ---
const account = ref(null);    // 钱包地址
const balance = ref(0);       // 余额
const chainId = ref(null);    // 网络ID
const chartRef = ref(null);   // ECharts DOM 引用
let myChart = null;           // ECharts 实例

// --- 计算属性 ---
// 1. 格式化地址 (显示前6位和后4位)
const shortAddress = computed(() => {
  return account.value ? `${account.value.slice(0, 6)}...${account.value.slice(-4)}` : '';
});

// 2. 动态按钮文字
const buttonText = computed(() => {
  if (!account.value) return "连接 Base 钱包";
  if (chainId.value !== 8453n) return "一键切换至 Base 网络";
  return "数据已同步";
});

// --- 核心方法 ---

// 初始化并连接 Base 网络
const connectBase = async () => {
  if (!window.ethereum) {
    alert("请安装 MetaMask 或 Coinbase Wallet!");
    return;
  }

  try {
    const provider = new ethers.BrowserProvider(window.ethereum);
    
    // A. 请求授权连接
    const accounts = await provider.send("eth_requestAccounts", []);
    account.value = accounts[0];

    // B. 检查并强制切换到 Base 链 (Chain ID: 8453)
    const network = await provider.getNetwork();
    chainId.value = network.chainId;

    if (chainId.value !== 8453n) {
      await switchNetwork();
    } else {
      await fetchBalance(provider);
    }
  } catch (error) {
    console.error("连接失败:", error);
  }
};

// 切换网络至 Base
const switchNetwork = async () => {
  try {
    await window.ethereum.request({
      method: 'wallet_switchEthereumChain',
      params: [{ chainId: '0x2105' }], // 8453 的十六进制
    });
    // 切换成功后刷新页面或重新获取数据
    window.location.reload();
  } catch (error) {
    // 如果钱包里没添加过 Base，则帮用户添加
    if (error.code === 4902) {
      await window.ethereum.request({
        method: 'wallet_addEthereumChain',
        params: [{
          chainId: '0x2105',
          chainName: 'Base Mainnet',
          nativeCurrency: { name: 'Ether', symbol: 'ETH', decimals: 18 },
          rpcUrls: ['https://mainnet.base.org'],
          blockExplorerUrls: ['https://basescan.org']
        }]
      });
    }
  }
};

// 获取 Base 链余额
const fetchBalance = async (provider) => {
  const rawBalance = await provider.getBalance(account.value);
  balance.value = parseFloat(ethers.formatEther(rawBalance)).toFixed(4);
  updateChart();
};

// --- ECharts 图表逻辑 ---
const initChart = () => {
  if (chartRef.value) {
    myChart = echarts.init(chartRef.value);
    const option = {
      tooltip: { trigger: 'item' },
      series: [{
        name: '资产分布',
        type: 'pie',
        radius: ['60%', '80%'],
        avoidLabelOverlap: false,
        itemStyle: { borderRadius: 10, borderColor: '#fff', borderWidth: 2 },
        label: { show: false },
        data: [
          { value: balance.value, name: 'Base ETH', itemStyle: { color: '#0052FF' } },
          { value: 0.0005, name: 'Gas 预留', itemStyle: { color: '#E2E8F0' } }
        ]
      }]
    };
    myChart.setOption(option);
  }
};

const updateChart = () => {
  if (myChart) {
    myChart.setOption({
      series: [{ data: [
        { value: balance.value, name: 'Base ETH', itemStyle: { color: '#0052FF' } },
        { value: 0.0005, name: 'Gas 预留', itemStyle: { color: '#E2E8F0' } }
      ]}]
    });
  }
};

// --- 生命周期与监听 ---
onMounted(() => {
  initChart();
  // 如果已经连接过，自动检测网络
  if (window.ethereum && window.ethereum.selectedAddress) {
    connectBase();
  }
});

// 监听余额变化更新图表
watch(balance, () => updateChart());
</script>

<style scoped>
.app-container {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  background-color: #f8fafc;
}

.dashboard-card {
  background: white;
  padding: 2.5rem;
  border-radius: 24px;
  box-shadow: 0 10px 25px rgba(0, 0, 0, 0.05);
  width: 100%;
  max-width: 400px;
  text-align: center;
}

.header-section {
  display: flex;
  align-items: center; /* 垂直对齐：让圆圈和文字对齐 */
  justify-content: center; /* 水平居中 */
  margin-bottom: 1.8rem; /* 与下方提示区的间距 */
  margin-top: -10px; /* 稍微往上移一点点，视觉更平衡 */
}

.base-circle {
  width: 24px; /* 减小圆圈，使其更优雅 */
  height: 24px;
  background-color: #0052FF; /* 使用 Base 官方蓝色 */
  border-radius: 50%; /* 变成正圆 */
  margin-right: 12px; /* 与文字的横向间距 */
  box-shadow: 0 0 10px rgba(0, 82, 255, 0.2); /* 淡淡的蓝光，更有科技感 */
}

.title-text {
  color: #1e293b; /* 深灰蓝色，看起来更高级 */
  font-size: 1.6rem; /* 调整字号，视觉更平衡 */
  font-weight: 800; /* 加粗 */
  margin: 0; /* 清除默认边距，保证对齐 */
  letter-spacing: -0.5px; /* 紧凑一点，Web3 流行风格 */
}

.status-box { margin-bottom: 1.5rem; padding: 1rem; background: #f1f5f9; border-radius: 12px; }
.address-tag { font-family: monospace; font-size: 0.85rem; color: #64748b; margin-top: 5px; }

.text-warning { color: #f59e0b; font-weight: 600; }
.text-danger { color: #ef4444; font-weight: 600; }
.text-success { color: #10b981; font-weight: 600; }

.btn-base {
  width: 100%;
  background: #0052FF;
  color: white;
  border: none;
  padding: 1rem;
  border-radius: 12px;
  font-size: 1.1rem;
  font-weight: 700;
  cursor: pointer;
  transition: all 0.2s;
}

.btn-base:hover { transform: translateY(-2px); box-shadow: 0 4px 12px rgba(0, 82, 255, 0.3); }
.btn-warning { background: #f59e0b; }

.chart-section { margin-top: 2rem; }
.echarts-dom { width: 100%; height: 250px; }

.balance-display { margin-top: 1rem; font-size: 1.2rem; }
.balance-display .value { font-weight: 800; color: #0052FF; margin-left: 8px; }

.footer { margin-top: 2rem; font-size: 0.8rem; color: #94a3b8; }
</style>