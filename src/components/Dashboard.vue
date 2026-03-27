<template>
  <div class="page-background">
    <div v-if="isInitializing" class="init-loader">
      <div class="spinner"></div>
      <p>Syncing Web3 Portal...</p>
    </div>

    <div v-else class="dashboard-card">
      <div class="header">
        <div class="title-group">
          <span class="dot" :class="{ 'dot-online': address }"></span>
          <h1>{{ address ? dynamicWalletTitle : 'Wallet Assets Dashboard' }}</h1>
        </div>
        <div class="mode-tag">Read-only Mode</div>
      </div>

      <div v-if="address" class="network-selector-wrap" @click="showChainModal = true">
        <div class="current-net-display">
          <div class="net-info-left">
            <span class="net-label">Network:</span>
            <span class="net-name">{{ currentChainName }}</span>
          </div>
          <span class="chevron">▼</span>
        </div>
      </div>

      <div v-if="!address" class="content-fade">
        <div class="connect-prompt-yellow">
          👋 欢迎！请连接钱包以查看资产余额
        </div>
        <button @click="openWalletModal" class="btn-main-solid">连接钱包</button>
      </div>

      <div v-else class="content-fade connected-inner">
        <div class="asset-display-box">
          <p class="status-tag-green">✅ 已同步 {{ currentChainName }} 生态数据</p>
          <p class="addr-text">{{ shortAddress }}</p>
          <div class="action-area">
            <span @click="disconnectWallet" class="link-red-text">断开连接</span>
          </div>
          
          <!-- <div id="main-chart"></div> -->
           <div class="chart-relative-wrap">
            <div v-if="isChartLoading" class="chart-loader">
              <div class="mini-spinner"></div>
              <span class="loading-text">Loading Data...</span>
            </div>
            <div id="main-chart" :class="{ 'chart-dim': isChartLoading }"></div>
          </div>

          <div class="balance-wrap">
            <!-- 当前余额: <span class="eth-val">{{ balance }} ETH</span> -->
             当前余额: 
            <span v-if="isChartLoading" class="eth-val-fetching">Fetching...</span>
            <span v-else class="eth-val">{{ balance }} ETH</span>
          </div>
        </div>
      </div>

      <div class="footer">
        <p class="disclaimer">🔐 安全预览模式</p>
        <p class="author">Built by WindyMD @ Multi-Chain</p>
      </div>
    </div>

    <transition name="fade">
      <div v-if="showChainModal || showWalletModal" class="modal-mask">
        <div class="modal-box">
          <div class="close-icon-wrap" @click="closeModals">
            <span class="close-icon-line line-1"></span>
            <span class="close-icon-line line-2"></span>
          </div>

          <template v-if="showChainModal">
            <h3 class="modal-title">切换网络</h3>
            <div class="list-container">
              <div v-for="(info, id) in SUPPORTED_CHAINS" :key="id" 
                   class="list-item" 
                   :class="{ 'active-selected': isSameChain(currentChainId, id) }"
                   @click="switchNetwork(id)">
                <span class="chain-dot" :style="{ background: info.color }"></span>
                <span class="item-name">{{ info.name }}</span>
                <span v-if="isSameChain(currentChainId, id)" class="check-mark">✓</span>
              </div>
            </div>
          </template>

          <template v-if="showWalletModal">
            <h3 class="modal-title">选择钱包</h3>
            <div class="list-container">
              <div v-for="wallet in detectedWallets" :key="wallet.name" 
                   class="list-item" @click="connectToWallet(wallet)">
                <img :src="wallet.icon" class="wallet-icon-sm" />
                <span class="item-name">{{ wallet.name }}</span>
              </div>
            </div>
          </template>
        </div>
      </div>
    </transition>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, nextTick } from 'vue';
import { ethers } from 'ethers';
import * as echarts from 'echarts';

const SUPPORTED_CHAINS = {
  '0x1': { name: 'Ethereum', color: '#627EEA' },
  '0x2105': { name: 'Base', color: '#0052ff' },
  '0xa4b1': { name: 'Arbitrum', color: '#28A0F0' },
  '0xa': { name: 'Optimism', color: '#FF0420' }
};

const address = ref('');
const balance = ref('0.0000');
const currentChainId = ref('');
const isInitializing = ref(true);
const showWalletModal = ref(false);
const showChainModal = ref(false);
const detectedWallets = ref([]);
const activeProvider = ref(null);
const currentWalletName = ref('Wallet');
let chart = null;
// ... 其他 ref 定义
const isChartLoading = ref(false); // 🌟 加载状态

const isSameChain = (id1, id2) => {
  if (!id1 || !id2) return false;
  return parseInt(id1.toString(), 16) === parseInt(id2.toString(), 16);
};

const currentChainName = computed(() => {
  const matchKey = Object.keys(SUPPORTED_CHAINS).find(key => isSameChain(key, currentChainId.value));
  return SUPPORTED_CHAINS[matchKey]?.name || 'Ethereum';
});

const shortAddress = computed(() => address.value ? `${address.value.slice(0, 6)}...${address.value.slice(-4)}` : '');
const dynamicWalletTitle = computed(() => `${currentWalletName.value} Assets`);

// const updateBalance = async () => {
//   if (!activeProvider.value || !address.value) return;
//   try {
//     const browserProvider = new ethers.BrowserProvider(activeProvider.value);
//     const rawBalance = await browserProvider.getBalance(address.value);
//     balance.value = parseFloat(ethers.formatEther(rawBalance)).toFixed(4);
//     initChart();
//   } catch (e) { balance.value = '0.0000'; }
// };

const updateBalance = async () => {
  if (!activeProvider.value || !address.value) return;
  
  isChartLoading.value = true; // 🌟 开始加载
  try {
    const browserProvider = new ethers.BrowserProvider(activeProvider.value);
    const rawBalance = await browserProvider.getBalance(address.value);
    balance.value = parseFloat(ethers.formatEther(rawBalance)).toFixed(4);
    
    // 渲染图表
    await initChart();
  } catch (e) {
    balance.value = '0.0000';
  } finally {
    // 稍微延迟关闭，防止闪烁，提升手感
    setTimeout(() => {
      isChartLoading.value = false; // 🌟 结束加载
    }, 400);
  }
};

// const switchNetwork = async (chainId) => {
//   if (!activeProvider.value) return;
//   try {
//     const targetId = chainId.toLowerCase();
//     await activeProvider.value.request({
//       method: 'wallet_switchEthereumChain',
//       params: [{ chainId: targetId }],
//     });
//     currentChainId.value = targetId;
//     showChainModal.value = false;
//     await updateBalance();
//   } catch (error) { console.warn("Switch failed", error); }
// };

// 切换网络时也确保触发
const switchNetwork = async (chainId) => {
  if (!activeProvider.value) return;
  try {
    const targetId = chainId.toLowerCase();
    await activeProvider.value.request({
      method: 'wallet_switchEthereumChain',
      params: [{ chainId: targetId }],
    });
    currentChainId.value = targetId;
    showChainModal.value = false;
    
    await updateBalance(); // 这里会触发上面的 isChartLoading 逻辑
  } catch (error) {
    console.warn("Switch failed", error);
  }
};

// 🌟 修改：连接时默认导向 Base 网络
const connectToWallet = async (wallet) => {
  activeProvider.value = wallet.provider;
  try {
    const accounts = await wallet.provider.request({ method: 'eth_requestAccounts' });
    address.value = accounts[0];
    currentWalletName.value = wallet.name;
    
    // 获取当前钱包所在的网络
    const initialId = await wallet.provider.request({ method: 'eth_chainId' });
    
    // 逻辑：如果当前不是 Base (0x2105)，则尝试切换到 Base
    const baseId = '0x2105';
    if (!isSameChain(initialId, baseId)) {
      try {
        await wallet.provider.request({
          method: 'wallet_switchEthereumChain',
          params: [{ chainId: baseId }],
        });
        currentChainId.value = baseId;
      } catch (switchError) {
        // 如果用户拒绝切换，则保留原网络
        currentChainId.value = initialId;
      }
    } else {
      currentChainId.value = initialId;
    }

    showWalletModal.value = false;
    await updateBalance();

    wallet.provider.on('chainChanged', (newId) => {
      currentChainId.value = newId;
      updateBalance(); 
    });
  } catch (e) { console.error("Connect failed"); }
};

const openWalletModal = () => {
  const wallets = [];
  const icons = { 'MetaMask': 'https://upload.wikimedia.org/wikipedia/commons/3/36/MetaMask_Fox.svg', 'Coinbase': 'https://avatars.githubusercontent.com/u/18060234?s=280&v=4' };
  if (window.ethereum?.providers?.length) {
    window.ethereum.providers.forEach(p => {
      const name = p.isMetaMask ? 'MetaMask' : p.isCoinbaseWallet ? 'Coinbase' : 'Browser';
      wallets.push({ name, provider: p, icon: icons[name] || '' });
    });
  } else if (window.ethereum) {
    const name = window.ethereum.isMetaMask ? 'MetaMask' : 'Wallet';
    wallets.push({ name, provider: window.ethereum, icon: icons[name] || '' });
  }
  detectedWallets.value = wallets;
  showWalletModal.value = true;
};

const disconnectWallet = () => { address.value = ''; balance.value = '0.0000'; };
const closeModals = () => { showWalletModal.value = false; showChainModal.value = false; };

const initChart = async () => {
  await nextTick();
  const chartDom = document.getElementById('main-chart');
  if (!chartDom) return;
  if (chart) chart.dispose();
  chart = echarts.init(chartDom);
  chart.setOption({
    tooltip: {
      trigger: 'item',
      backgroundColor: 'rgba(15, 23, 42, 0.9)',
      textStyle: { color: '#fff', fontSize: 12, fontWeight: 'bold' },
      formatter: (params) => `${params.name}: <b>${params.value} ETH</b>`
    },
    series: [{
      type: 'pie', radius: ['65%', '85%'],
      avoidLabelOverlap: false,
      itemStyle: { borderRadius: 8, borderColor: '#fff', borderWidth: 2 },
      data: [
        { value: parseFloat(balance.value) || 0, name: currentChainName.value, itemStyle: { color: '#0052ff' } },
        { value: 0.0001, name: 'Others', itemStyle: { color: '#f8fafc' } }
      ],
      label: { show: false }
    }]
  });
};

onMounted(() => { setTimeout(() => isInitializing.value = false, 800); });
</script>

<style scoped>
/* 严格保留你的精致弹窗与下拉框样式 */
.page-background { display: flex; justify-content: center; align-items: center; min-height: 100vh; background: #f8fafc; font-family: -apple-system, sans-serif; }
.dashboard-card { background: white; padding: 32px; border-radius: 28px; box-shadow: 0 20px 60px rgba(0,0,0,0.06); width: 380px; }

.modal-mask { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(15, 23, 42, 0.4); backdrop-filter: blur(4px); display: flex; justify-content: center; align-items: center; z-index: 1000; }
.modal-box { background: white; padding: 24px; border-radius: 24px; width: 320px; position: relative; }
.modal-title { font-size: 1.05rem; font-weight: 800; color: #1e293b; margin-bottom: 18px; text-align: center; }

.network-selector-wrap { margin-bottom: 20px; cursor: pointer; }
.current-net-display { background: #f8fafc; padding: 12px 16px; border-radius: 14px; display: flex; justify-content: space-between; align-items: center; font-size: 13px; border: 1px solid #e2e8f0; }
.net-info-left { display: flex; gap: 8px; }
.net-label { color: #94a3b8; font-weight: 500; }
.net-name { color: #1e293b; font-weight: 700; }
.chevron { font-size: 10px; color: #94a3b8; }

.connect-prompt-yellow { background: #fffbeb; color: #b45309; padding: 18px; border-radius: 16px; text-align: center; margin-bottom: 25px; font-size: 14px; font-weight: 600; border: 1px solid #fef3c7; }
.btn-main-solid { background: #0052ff; color: white; width: 100%; padding: 16px; border: none; border-radius: 14px; font-weight: 700; cursor: pointer; }

.list-item { display: flex; align-items: center; gap: 10px; padding: 12px; border-radius: 12px; border: 1px solid #e2e8f0; margin-top: 8px; cursor: pointer; transition: 0.2s; }
.active-selected { border-color: #0052ff; background: #f0f7ff; }
.wallet-icon-sm { width: 24px; height: 24px; }
.chain-dot { width: 8px; height: 8px; border-radius: 50%; }
.item-name { font-size: 14px; font-weight: 600; color: #334155; }
.check-mark { color: #0052ff; font-weight: 900; margin-left: auto; }

.close-icon-wrap { position: absolute; top: 16px; right: 16px; width: 28px; height: 28px; cursor: pointer; display: flex; align-items: center; justify-content: center; }
.close-icon-line { position: absolute; width: 14px; height: 2px; background: #94a3b8; }
.line-1 { transform: rotate(45deg); }
.line-2 { transform: rotate(-45deg); }

.header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 25px; }
.title-group h1 { font-size: 1.1rem; font-weight: 800; color: #1e293b; margin: 0; }
.dot { width: 8px; height: 8px; border-radius: 50%; background: #cbd5e0; }
.dot-online { background: #0052ff; box-shadow: 0 0 8px rgba(0, 82, 255, 0.4); }
.mode-tag { font-size: 10px; background: #f1f5f9; color: #64748b; padding: 4px 10px; border-radius: 20px; font-weight: 600; }

.asset-display-box { background: #f0fdf4; padding: 25px; border-radius: 22px; text-align: center; }
.addr-text { color: #64748b; font-family: monospace; font-size: 13px; margin: 8px 0; }
.link-red-text { color: #ef4444; font-size: 12px; text-decoration: underline; cursor: pointer; }

/* 🌟 加载状态容器样式 */
.chart-relative-wrap {
  position: relative;
  width: 100%;
  height: 200px;
}

.chart-loader {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 10;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  background: rgba(240, 253, 244, 0.6); /* 匹配你资产框的绿色调背景 */
  backdrop-filter: blur(2px); /* 淡淡的毛玻璃 */
  border-radius: 22px;
}

.mini-spinner {
  width: 24px;
  height: 24px;
  border: 3px solid #e2e8f0;
  border-top-color: #0052ff; /* 品牌蓝 */
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
  margin-bottom: 8px;
}

.loading-text {
  font-size: 10px;
  color: #64748b;
  font-weight: 600;
  letter-spacing: 0.5px;
}

/* 图表正在加载时的变淡效果 */
.chart-dim {
  opacity: 0.2;
  transition: opacity 0.3s ease;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

#main-chart { width: 100%; height: 200px; }
/* 🌟 新增：Fetching 占位符样式 */
.eth-val-fetching {
  color: #94a3b8; /* Fetching 使用灰色，满足截图18需求 */
  font-weight: 700;
  font-size: 1.1rem;
  letter-spacing: 0.5px;
  margin-left: 5px;
}
.eth-val { color: #0052ff; font-weight: 900; font-size: 1.5rem; }
.footer { margin-top: 25px; border-top: 1px solid #f1f5f9; padding-top: 15px; text-align: center; font-size: 11px; color: #94a3b8; }
</style>