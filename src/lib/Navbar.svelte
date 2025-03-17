<script lang="ts">
  import { onMount } from 'svelte';
  import Onboard from '@web3-onboard/core';
  import injectedModule from '@web3-onboard/injected-wallets';

  export let wallet: any;
  export let onboard: any;

  let connecting = false;
  let wrongNetwork = false;

  const PULSECHAIN_ID = '0x171';

  async function checkNetwork() {
    if (!wallet) return;
    
    const chainId = wallet.chains[0].id;
    wrongNetwork = chainId !== PULSECHAIN_ID;
    
    if (wrongNetwork) {
      try {
        await onboard.setChain({ chainId: PULSECHAIN_ID });
        wrongNetwork = false;
      } catch (error) {
        console.error('Failed to switch network:', error);
      }
    }
  }

  async function connectWallet() {
    connecting = true;
    try {
      const wallets = await onboard.connectWallet();
      if (wallets[0]) {
        wallet = wallets[0];
        await checkNetwork();
      }
    } catch (error) {
      console.error('Error connecting wallet:', error);
    }
    connecting = false;
  }

  // Subscribe to chain changes
  $: if (wallet) {
    wallet.provider.on('chainChanged', () => {
      checkNetwork();
    });
  }

  onMount(() => {
    return () => {
      if (wallet?.provider) {
        wallet.provider.removeAllListeners();
      }
    };
  });
</script>

<nav class="bg-[#1c1f2a] py-4 px-6 shadow-lg">
  <div class="max-w-7xl mx-auto flex justify-between items-center">
    <div class="flex items-center">
      <img src="https://raw.githubusercontent.com/jaxnalia/lean-frontend-master/refs/heads/main/src/lib/images/lean.png" alt="Logo" class="h-8 w-8" />
      <span class="ml-2 text-xl font-bold text-white">LEAN Farm</span>
    </div>
    
    {#if wrongNetwork}
      <div class="flex items-center">
        <span class="text-red-500 mr-4">Wrong Network</span>
        <button
          class="btn-primary"
          on:click={checkNetwork}
        >
          Switch to PulseChain
        </button>
      </div>
    {:else}
      <button
        class="btn-primary {connecting ? 'opacity-75 cursor-not-allowed' : ''}"
        on:click={connectWallet}
        disabled={connecting}
      >
        {#if connecting}
          Connecting...
        {:else if wallet}
          {wallet.accounts[0].address.slice(0, 6)}...{wallet.accounts[0].address.slice(-4)}
        {:else}
          Connect Wallet
        {/if}
        
      </button>
    {/if}
  
  </div>
</nav>