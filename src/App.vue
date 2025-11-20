<script setup>
import { ref } from 'vue';
import { ethers } from 'ethers';
import { ABI } from './abi';

const CONTRACT_ADDRESS = '0x4a5327347f077e72d2aab19f68ba8a7f12ec5d63';
const DEFAULT_PROPOSAL_ID = '97018566875849739057791829757781237329950408521603525975301422956278121248932';

const GNOSIS_CHAIN_ID = '0x64'; // Chain ID 100

const proposalId = ref(DEFAULT_PROPOSAL_ID);
const support = ref(null); // 0: Against, 1: For, 2: Abstain
const reason = ref('');
const account = ref(null);
const status = ref('');
const loading = ref(false);
const isWrongNetwork = ref(false);

const checkNetwork = async () => {
  if (window.ethereum) {
    const chainId = await window.ethereum.request({ method: 'eth_chainId' });
    isWrongNetwork.value = chainId !== GNOSIS_CHAIN_ID;
  }
};

const switchNetwork = async () => {
  try {
    await window.ethereum.request({
      method: 'wallet_switchEthereumChain',
      params: [{ chainId: GNOSIS_CHAIN_ID }],
    });
    isWrongNetwork.value = false;
  } catch (switchError) {
    // This error code indicates that the chain has not been added to MetaMask.
    if (switchError.code === 4902) {
      try {
        await window.ethereum.request({
          method: 'wallet_addEthereumChain',
          params: [
            {
              chainId: GNOSIS_CHAIN_ID,
              chainName: 'Gnosis',
              rpcUrls: ['https://rpc.gnosischain.com'],
              blockExplorerUrls: ['https://gnosisscan.io'],
              nativeCurrency: {
                name: 'xDAI',
                symbol: 'xDAI',
                decimals: 18,
              },
            },
          ],
        });
        isWrongNetwork.value = false;
      } catch (addError) {
        status.value = 'Failed to add Gnosis Chain: ' + addError.message;
      }
    } else {
      status.value = 'Failed to switch network: ' + switchError.message;
    }
  }
};

const connectWallet = async () => {
  if (typeof window.ethereum !== 'undefined') {
    try {
      const accounts = await window.ethereum.request({ method: 'eth_requestAccounts' });
      account.value = accounts[0];
      await checkNetwork();
      
      window.ethereum.on('chainChanged', (chainId) => {
        isWrongNetwork.value = chainId !== GNOSIS_CHAIN_ID;
      });
      
      status.value = 'Wallet connected';
    } catch (error) {
      status.value = 'Failed to connect wallet: ' + error.message;
    }
  } else {
    status.value = 'Please install MetaMask!';
  }
};

const vote = async () => {
  if (!account.value) {
    status.value = 'Please connect wallet first';
    return;
  }
  
  if (isWrongNetwork.value) {
    await switchNetwork();
    if (isWrongNetwork.value) return; // Abort if still on wrong network
  }

  if (support.value === null) {
    status.value = 'Please select a vote option';
    return;
  }

  loading.value = true;
  status.value = 'Initiating vote...';

  try {
    const provider = new ethers.BrowserProvider(window.ethereum);
    const signer = await provider.getSigner();
    const contract = new ethers.Contract(CONTRACT_ADDRESS, ABI, signer);

    // STRICT ENFORCEMENT: Ensure support is an integer 0, 1, or 2
    const supportValue = parseInt(support.value);
    if (![0, 1, 2].includes(supportValue)) {
      throw new Error("Invalid vote option. Must be 0 (Against), 1 (For), or 2 (Abstain).");
    }

    let tx;
    if (reason.value.trim()) {
      console.log(`Voting with reason. Proposal: ${proposalId.value}, Support: ${supportValue}, Reason: ${reason.value}`);
      tx = await contract.castVoteWithReason(proposalId.value, supportValue, reason.value);
    } else {
      console.log(`Voting without reason. Proposal: ${proposalId.value}, Support: ${supportValue}`);
      tx = await contract.castVote(proposalId.value, supportValue);
    }

    status.value = 'Transaction sent: ' + tx.hash;
    await tx.wait();
    status.value = 'Vote confirmed!';
  } catch (error) {
    console.error(error);
    status.value = 'Error: ' + (error.reason || error.message);
  } finally {
    loading.value = false;
  }
};
</script>

<template>
  <div class="container">
    <h1>RealToken DAO Voting</h1>
    
    <div v-if="!account" class="connect-section">
      <button @click="connectWallet" class="btn-primary">Connect Wallet</button>
    </div>
    
    <div v-else class="voting-section">
      <p class="account-info">Connected: {{ account }}</p>
      
      <div v-if="isWrongNetwork" class="network-warning">
        ⚠️ You are on the wrong network. 
        <button @click="switchNetwork" class="btn-small">Switch to Gnosis Chain</button>
      </div>
      
      <div class="form-group">
        <label>Proposal ID:</label>
        <input v-model="proposalId" type="text" class="input-field" />
      </div>

      <div class="form-group">
        <label>Vote:</label>
        <div class="radio-group">
          <label class="radio-label">
            <input type="radio" :value="1" v-model="support" />
            <span class="vote-for">FOR</span>
          </label>
          <label class="radio-label">
            <input type="radio" :value="0" v-model="support" />
            <span class="vote-against">AGAINST</span>
          </label>
          <label class="radio-label">
            <input type="radio" :value="2" v-model="support" />
            <span class="vote-abstain">ABSTAIN</span>
          </label>
        </div>
      </div>

      <div class="form-group">
        <label>Reason (Optional):</label>
        <textarea v-model="reason" class="textarea-field" placeholder="Why are you voting this way?"></textarea>
      </div>

      <button @click="vote" :disabled="loading" class="btn-primary">
        {{ loading ? 'Voting...' : 'Cast Vote' }}
      </button>
    </div>

    <div v-if="status" class="status-message">
      {{ status }}
    </div>
  </div>
</template>

<style>
:root {
  --bg-color: #1a1a1a;
  --card-bg: #2d2d2d;
  --text-color: #e0e0e0;
  --primary-color: #f2a900; /* RealToken Gold/Orange */
  --primary-hover: #d49400;
  --danger-color: #e74c3c;
  --success-color: #27ae60;
  --warning-bg: #3e3010;
  --warning-text: #f2a900;
  --input-bg: #383838;
  --border-color: #444;
}

body {
  background-color: var(--bg-color);
  color: var(--text-color);
  margin: 0;
  font-family: 'Inter', sans-serif;
}
</style>

<style scoped>
.container {
  max-width: 700px;
  margin: 2rem auto;
  padding: 2rem;
}

h1 {
  text-align: center;
  margin-bottom: 2.5rem;
  color: var(--text-color);
  font-weight: 600;
  letter-spacing: 0.5px;
}

.connect-section {
  display: flex;
  justify-content: center;
  margin-top: 4rem;
}

.voting-section {
  background-color: var(--card-bg);
  padding: 2rem;
  border-radius: 12px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.3);
}

.btn-primary {
  background-color: var(--primary-color);
  color: #121212; /* Dark text on gold button for contrast */
  border: none;
  padding: 12px 24px;
  border-radius: 8px;
  cursor: pointer;
  font-size: 1rem;
  font-weight: 600;
  width: 100%;
  transition: all 0.2s ease;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.btn-primary:hover {
  background-color: var(--primary-hover);
  transform: translateY(-1px);
}

.btn-primary:disabled {
  background-color: #555;
  color: #888;
  cursor: not-allowed;
  transform: none;
}

.form-group {
  margin-bottom: 1.5rem;
}

label {
  display: block;
  margin-bottom: 0.8rem;
  font-weight: 500;
  color: #bbb;
  font-size: 0.9rem;
  text-transform: uppercase;
  letter-spacing: 0.5px;
}

.input-field, .textarea-field {
  width: 100%;
  padding: 12px;
  background-color: var(--input-bg);
  border: 1px solid var(--border-color);
  border-radius: 6px;
  font-size: 1rem;
  color: var(--text-color);
  transition: border-color 0.2s;
  box-sizing: border-box;
}

.input-field:focus, .textarea-field:focus {
  outline: none;
  border-color: var(--primary-color);
}

.textarea-field {
  min-height: 100px;
  resize: vertical;
}

.radio-group {
  display: flex;
  gap: 1rem;
}

.radio-label {
  display: flex;
  align-items: center;
  gap: 0.8rem;
  cursor: pointer;
  padding: 12px;
  background-color: var(--input-bg);
  border: 1px solid var(--border-color);
  border-radius: 6px;
  flex: 1;
  justify-content: center;
  transition: all 0.2s;
}

.radio-label:hover {
  border-color: #666;
}

/* Custom Radio Styling */
input[type="radio"] {
  accent-color: var(--primary-color);
  transform: scale(1.2);
}

.vote-for { color: var(--success-color); font-weight: bold; }
.vote-against { color: var(--danger-color); font-weight: bold; }
.vote-abstain { color: #aaa; font-weight: bold; }

.status-message {
  margin-top: 2rem;
  padding: 1rem;
  background-color: var(--card-bg);
  border: 1px solid var(--border-color);
  border-radius: 8px;
  text-align: center;
  word-break: break-all;
  color: var(--text-color);
}

.account-info {
  text-align: center;
  font-size: 0.9rem;
  color: #888;
  margin-bottom: 2rem;
  font-family: monospace;
  background: rgba(0,0,0,0.2);
  padding: 8px;
  border-radius: 4px;
}

.network-warning {
  background-color: var(--warning-bg);
  color: var(--warning-text);
  padding: 12px;
  border-radius: 6px;
  margin-bottom: 1.5rem;
  text-align: center;
  border: 1px solid rgba(242, 169, 0, 0.3);
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 10px;
}

.btn-small {
  background-color: var(--primary-color);
  color: #121212;
  border: none;
  padding: 6px 12px;
  border-radius: 4px;
  cursor: pointer;
  font-size: 0.8rem;
  font-weight: 600;
}

.btn-small:hover {
  background-color: var(--primary-hover);
}
</style>
