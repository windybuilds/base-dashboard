<template>
  <div class="page-background">
    <div v-if="isInitializing" class="init-loader">
      <div class="spinner"></div>
      <p>Syncing Base Network...</p>
    </div>

    <div v-else class="dashboard-card">
      <div class="header">
        <div class="title-group">
          <span class="dot"></span>
          <h1>{{ address ? currentWalletTitle : 'Wallet Assets Dashboard' }}</h1>
        </div>
        <div class="mode-tag">Read-only Mode</div>
      </div>

      <div v-if="!address" class="content-fade connect-area">
        <div class="connect-prompt">
          <span>⚠️</span> 请先连接您的 Base 钱包
        </div>
        <button @click="openWalletModal" class="btn-main">连接钱包</button>
      </div>

      <div v-else class="content-fade connected-box">
        <p class="status-tag">✅ 已同步 Base 生态数据</p>
        <p class="addr-text">{{ shortAddress }}</p>
        <button @click="disconnectWallet" class="btn-link-red">断开连接</button>
        
        <div id="main-chart"></div>

        <div class="balance-wrap">
          当前余额: <span class="eth-val">{{ balance }} ETH</span>
        </div>
      </div>

      <transition name="fade">
        <div v-if="showWalletModal" class="modal-mask">
          <div class="modal-box">
            <div class="close-icon-wrap" @click="showWalletModal = false">
              <span class="close-icon-line line-1"></span>
              <span class="close-icon-line line-2"></span>
            </div>
            
            <h3 class="modal-title">选择钱包</h3>
            <div class="wallet-list">
              <div v-for="wallet in detectedWallets" 
                   :key="wallet.name" 
                   class="wallet-item" 
                   @click="connectToWallet(wallet)">
                <img :src="wallet.icon || defaultWalletIcon" :alt="wallet.name" />
                <span>{{ wallet.name }}</span>
              </div>
            </div>
          </div>
        </div>
      </transition>

      <div class="footer">
        <p class="disclaimer">🔐 仅读取余额，不请求转账权限</p>
        <p class="author">Built by WindyMD @ Base Ecosystem</p>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, nextTick } from 'vue';
import { ethers } from 'ethers';
import * as echarts from 'echarts';

const address = ref('');
const balance = ref('0.0000');
const isInitializing = ref(true); 
const showWalletModal = ref(false);
const detectedWallets = ref([]);
const currentWalletName = ref('Base ETH'); 
const currentWalletTitle = ref('Base'); 
let chart = null;

const staticIcons = {
  'MetaMask': 'https://upload.wikimedia.org/wikipedia/commons/3/36/MetaMask_Fox.svg',
  'Coinbase': 'https://avatars.githubusercontent.com/u/18060234?s=280&v=4'
};

const defaultWalletIcon = 'data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCAyNCAyNCIgZmlsbD0ibm9uZSIgc3Ryb2tlPSIjOTRhM2I4IiBzdHJva2Utd2lkdGg9IjIiPjxyZWN0IHg9IjMiIHk9IjUiIHdpZHRoPSIxOCIgaGVpZ2h0PSIxNCIgcng9IjMiLz48cGF0aCBkPSJNMTYgMTF2Mm0wIDRoMCIvPjwvc3ZnPg==';

const shortAddress = computed(() => address.value ? `${address.value.slice(0, 6)}...${address.value.slice(-4)}` : '');

const discoverWallets = () => {
  const wallets = [];
  if (window.ethereum?.providers?.length) {
    window.ethereum.providers.forEach(p => {
      const name = p.isMetaMask ? 'MetaMask' : p.isCoinbaseWallet ? 'Coinbase' : (p.name || 'Browser');
      wallets.push({ name, icon: staticIcons[name] || p.icon, provider: p });
    });
  } else if (window.ethereum) {
    const name = window.ethereum.isMetaMask ? 'MetaMask' : 'Base';
    wallets.push({ name, icon: staticIcons[name], provider: window.ethereum });
  }
  detectedWallets.value = wallets;
};

const openWalletModal = () => {
  discoverWallets();
  showWalletModal.value = true;
};

const initChart = async () => {
  await nextTick();
  setTimeout(() => {
    const chartDom = document.getElementById('main-chart');
    if (!chartDom) return;
    if (chart) chart.dispose();
    chart = echarts.init(chartDom);
    chart.setOption({
      tooltip: { trigger: 'item', formatter: '{b}: <b>{c}</b>' },
      series: [{
        type: 'pie',
        radius: ['65%', '85%'],
        itemStyle: { borderRadius: 8, borderColor: '#fff', borderWidth: 2 },
        data: [
          { value: parseFloat(balance.value) || 0, name: currentWalletName.value, itemStyle: { color: '#0052ff' } },
          { value: 0.0001, name: 'Other Assets', itemStyle: { color: '#f8fafc' } }
        ],
        label: { show: false }
      }]
    });
  }, 200);
};

const connectToWallet = async (wallet) => {
  try {
    const accounts = await wallet.provider.request({ method: 'eth_requestAccounts' });
    showWalletModal.value = false;
    currentWalletName.value = wallet.name + ' ETH';
    currentWalletTitle.value = wallet.name; 
    address.value = accounts[0];
    const provider = new ethers.BrowserProvider(wallet.provider);
    const rawBalance = await provider.getBalance(address.value);
    balance.value = parseFloat(ethers.formatEther(rawBalance)).toFixed(4);
    initChart();
  } catch (error) { console.error(error); }
};

const disconnectWallet = () => {
  address.value = '';
  balance.value = '0.0000';
  if (chart) { chart.dispose(); chart = null; }
};

onMounted(() => {
  setTimeout(() => { isInitializing.value = false; }, 800);
});
</script>

<style scoped>
/* 基础容器 */
.page-background { 
  display: flex; justify-content: center; align-items: center; 
  min-height: 100vh; background: #f8fafc; font-family: -apple-system, sans-serif; 
}

/* 主卡片 */
.dashboard-card { 
  background: white; padding: 32px; border-radius: 28px; 
  box-shadow: 0 20px 60px rgba(0,0,0,0.06); width: 380px; 
}

.header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 25px; }
.title-group { display: flex; align-items: center; gap: 8px; }
.dot { width: 9px; height: 9px; background: #0052ff; border-radius: 50%; }
.title-group h1 { font-size: 1.15rem; font-weight: 700; margin: 0; color: #1e293b; }
.mode-tag { background: #f1f5f9; color: #64748b; font-size: 11px; padding: 5px 12px; border-radius: 20px; font-weight: 500; }

/* 内容区 */
.connect-prompt { background: #fffbeb; color: #92400e; padding: 15px; border-radius: 12px; text-align: center; margin-bottom: 20px; font-size: 14px; }
.btn-main { background: #0052ff; color: white; width: 100%; padding: 16px; border: none; border-radius: 14px; font-weight: 600; cursor: pointer; font-size: 1rem; transition: 0.2s; }
.btn-main:hover { background: #0045d9; transform: translateY(-1px); }

.connected-box { background: #f0fdf4; padding: 20px; border-radius: 18px; text-align: center; margin-bottom: 20px; }
.addr-text { color: #64748b; font-family: monospace; margin: 8px 0; font-size: 13px; }
.btn-link-red { background: none; border: none; color: #ef4444; text-decoration: underline; cursor: pointer; font-size: 12px; }

#main-chart { width: 100%; height: 220px; margin: 10px 0; }
.eth-val { color: #0052ff; font-weight: 800; font-size: 1.4rem; }

/* 🌟 弹窗美化 */
.modal-mask { position: fixed; top: 0; left: 0; width: 100%; height: 100%; background: rgba(15, 23, 42, 0.5); backdrop-filter: blur(4px); display: flex; justify-content: center; align-items: center; z-index: 1000; }

.modal-box { 
  background: white; padding: 32px 24px; border-radius: 24px; 
  width: 330px; position: relative; box-shadow: 0 25px 50px -12px rgba(0, 0, 0, 0.25);
  animation: modalScale 0.3s ease-out;
}

@keyframes modalScale {
  from { opacity: 0; transform: scale(0.95); }
  to { opacity: 1; transform: scale(1); }
}

.modal-title { margin: 0 0 20px 0; font-size: 1.25rem; font-weight: 700; color: #1e293b; text-align: center; }

/* 🌟 右上角 X 按钮样式 */
.close-icon-wrap {
  position: absolute; top: 20px; right: 20px;
  width: 32px; height: 32px;
  cursor: pointer; border-radius: 50%;
  transition: all 0.2s;
  display: flex; align-items: center; justify-content: center;
}

.close-icon-wrap:hover { background: #f1f5f9; transform: rotate(90deg); }

.close-icon-line {
  position: absolute; width: 18px; height: 2px;
  background: #64748b; border-radius: 10px;
}

.line-1 { transform: rotate(45deg); }
.line-2 { transform: rotate(-45deg); }

/* 钱包列表项 */
.wallet-item { 
  display: flex; align-items: center; gap: 14px; padding: 16px; 
  border: 1px solid #e2e8f0; border-radius: 16px; cursor: pointer; 
  margin-top: 12px; transition: all 0.2s; 
}
.wallet-item:hover { border-color: #0052ff; background: #f0f7ff; transform: scale(1.02); }
.wallet-item img { width: 30px; height: 30px; object-fit: contain; }
.wallet-item span { font-weight: 600; color: #334155; }

.footer { margin-top: 25px; border-top: 1px solid #f1f5f9; padding-top: 15px; text-align: center; font-size: 11px; color: #94a3b8; }
</style>