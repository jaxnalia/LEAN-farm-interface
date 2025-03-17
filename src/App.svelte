<script lang="ts">
  import FarmCard from './lib/FarmCard.svelte';
  import Navbar from './lib/Navbar.svelte';
  import { onMount } from 'svelte';
  import Onboard from '@web3-onboard/core';
  import injectedModule from '@web3-onboard/injected-wallets';

  let wallet: any = null;

  const farms = [
    {
      pair: 'LEAN-PLSX',
      apr: 125.4,
      tvl: 1234567,
      earned: 0.45,
      staked: 123.45,
      logo1: 'https://raw.githubusercontent.com/jaxnalia/lean-frontend-master/refs/heads/main/src/lib/images/lean.png',
      logo2: 'https://raw.githubusercontent.com/jaxnalia/lean-frontend-master/refs/heads/main/src/lib/images/plsx.png'
    },
    {
      pair: 'LEAN-DAI',
      apr: 89.2,
      tvl: 987654,
      earned: 0.12,
      staked: 45.67,
      logo1: 'https://raw.githubusercontent.com/jaxnalia/lean-frontend-master/refs/heads/main/src/lib/images/lean.png',
      logo2: 'https://cryptologos.cc/logos/multi-collateral-dai-dai-logo.png'
    },
    {
      pair: 'LEAN-LIT',
      apr: 76.5,
      tvl: 456789,
      earned: 0.89,
      staked: 67.89,
      logo1: 'https://raw.githubusercontent.com/jaxnalia/lean-frontend-master/refs/heads/main/src/lib/images/lean.png',
      logo2: 'https://raw.githubusercontent.com/jaxnalia/lean-frontend-master/refs/heads/main/src/lib/images/lit.png'
    }
  ];

  const injected = injectedModule();
  const onboard = Onboard({
    wallets: [injected],
    chains: [
      {
        id: '0x171',
        token: 'PLS',
        label: 'PulseChain',
        rpcUrl: 'https://rpc.pulsechain.com'
      }
    ],
    appMetadata: {
      name: 'LEAN Farm',
      icon: 'https://www.gopulsechain.com/files/LogoVector.svg',
      description: 'LEAN Farming Interface'
    }
  });

  onboard.state.select('wallets').subscribe((wallets) => {
    if (wallets.length > 0) {
      wallet = wallets[0];
    } else {
      wallet = null;
    }
  });
</script>

<Navbar {wallet} {onboard} />

<main class="min-h-screen py-8 px-4 md:px-8">
  <div class="max-w-4xl mx-auto">
    <div class="text-center mb-12">
      <h1 class="text-4xl font-bold mb-4">Farms</h1>
      <p class="text-gray-400">Stake LP tokens to earn PLS rewards</p>
    </div>

    <div class="grid gap-4">
      {#each farms as farm}
        <FarmCard {...farm} {wallet} />
      {/each}
      
    </div>
  </div>
</main>