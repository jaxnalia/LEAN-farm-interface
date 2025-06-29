<script lang="ts">
  import FarmCard from './lib/FarmCard.svelte';
  import Navbar from './lib/Navbar.svelte';
  import { onMount } from 'svelte';
  import Onboard from '@web3-onboard/core';
  import injectedModule from '@web3-onboard/injected-wallets';
  import { ethers } from 'ethers';
  import { Toaster } from 'svelte-french-toast';

  let wallet: any = null;
  let totalAllocPoint = 0;
  let farmWeights: { [key: string]: number } = {};
  let userEarned: { [key: string]: string } = {};

  const MASTERCHEF_ADDRESS = '0xbE7f4fFfDe4241cA25eb27616aE3974aF0a023fD';
  const MASTERCHEF_ABI = [
    "function totalAllocPoint() view returns (uint256)",
    "function poolInfo(uint256) view returns (address lpToken, uint256 allocPoint, uint256 lastRewardBlock, uint256 accTokenPerShare)",
    "function pendingTokens(uint256 _pid, address _user) view returns (uint256)"
  ];

  const LP_TOKEN_ABI = [
    "function balanceOf(address owner) view returns (uint256)",
    "function getReserves() view returns (uint112 reserve0, uint112 reserve1, uint32 blockTimestampLast)",
    "function totalSupply() view returns (uint256)"
  ];

   // LEAN/WPLS pair address for price calculation
  const LEAN_WPLS_PAIR = '0x9961c2652B301c4A25256Db05316d2be11CEbaB1';

  // Token prices in USD - will be updated from CoinGecko and pair ratios
  const TOKEN_PRICES = {
    'LEAN': 0.000000,  // Will be calculated from LEAN/WPLS pair
    'PLSX': 0.00000000,  // Will be updated from CoinGecko
    'LIT': 0.0000079,   // Static price for now
    'WPLS': 0.00000     // Will be updated from CoinGecko
  };

  // Reward constants
  const BLOCKS_PER_YEAR = 31536000 / 3; // 3s block time (31536000 seconds per year)
  const REWARD_PER_BLOCK = ethers.BigNumber.from('10000000000000000000'); // 10 tokens per block

  let farms = [
    {
      pair: 'LEAN-PLSX',
      tvl: 0,
      apr: 0,
      earned: 0,
      staked: 0,
      logo1: 'https://raw.githubusercontent.com/jaxnalia/lean-frontend-master/refs/heads/main/src/lib/images/lean.png',
      logo2: 'https://raw.githubusercontent.com/jaxnalia/lean-frontend-master/refs/heads/main/src/lib/images/plsx.png',
      poolId: 0,
      lpAddress: '0x9A5C8868a7B6c099238A87b7e4d0AE03f9CB4393'
    },
    {
      pair: 'LIT-PLSX',
      tvl: 0,
      apr: 0,
      earned: 0,
      staked: 0,
      logo1: 'https://raw.githubusercontent.com/jaxnalia/lean-frontend-master/refs/heads/main/src/lib/images/lit.png',
      logo2: 'https://raw.githubusercontent.com/jaxnalia/lean-frontend-master/refs/heads/main/src/lib/images/plsx.png',
      poolId: 1,
      lpAddress: '0xb75443509d897cE6D9b8F67C4B9eCc45dDdf3160'
    },
    {
      pair: 'LIT-LEAN',
      tvl: 0,
      apr: 0,
      earned: 0,
      staked: 0,
      logo1: 'https://raw.githubusercontent.com/jaxnalia/lean-frontend-master/refs/heads/main/src/lib/images/lit.png',
      logo2: 'https://raw.githubusercontent.com/jaxnalia/lean-frontend-master/refs/heads/main/src/lib/images/lean.png',
      poolId: 2,
      lpAddress: '0x567499ec3428c77F8B36bc6cA5221961330228f6'
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

  async function fetchTokenPrices() {
    try {
      // Fetch PLSX and WPLS prices from CoinGecko
      const response = await fetch('https://api.coingecko.com/api/v3/simple/price?ids=pulsechain,pulsex&vs_currencies=usd');
      const data = await response.json();
      
      if (data.pulsechain?.usd) {
        TOKEN_PRICES.WPLS = data.pulsechain.usd;
      }
      
      if (data.pulsex?.usd) {
        TOKEN_PRICES.PLSX = data.pulsex.usd;
      }
      
      // Calculate LEAN price from LEAN/WPLS pair
      await calculateLeanPrice();
      
      console.log('Updated token prices:', TOKEN_PRICES);
    } catch (error) {
      console.error('Error fetching token prices from CoinGecko:', error);
      // Keep using fallback prices if API fails
    }
  }

  async function calculateLeanPrice() {
    try {
      const provider = new ethers.providers.JsonRpcProvider('https://rpc.pulsechain.com');
      const leanWplsPair = new ethers.Contract(LEAN_WPLS_PAIR, LP_TOKEN_ABI, provider);
      
      // Get reserves from LEAN/WPLS pair
      const reserves = await leanWplsPair.getReserves();
      
      // Assuming LEAN is token0 and WPLS is token1 (you may need to verify this)
      // If reversed, swap reserve0 and reserve1 in the calculation
      const leanReserve = ethers.utils.formatEther(reserves[0]); // LEAN reserve
      const wplsReserve = ethers.utils.formatEther(reserves[1]); // WPLS reserve
      
      // Calculate LEAN price in WPLS terms
      const leanPriceInWpls = parseFloat(wplsReserve) / parseFloat(leanReserve);
      
      // Calculate LEAN price in USD
      TOKEN_PRICES.LEAN = leanPriceInWpls * TOKEN_PRICES.WPLS;
      
      console.log(`LEAN/WPLS ratio: ${leanPriceInWpls}, LEAN USD price: $${TOKEN_PRICES.LEAN}`);
    } catch (error) {
      console.error('Error calculating LEAN price from pair:', error);
      // Keep using fallback LEAN price if calculation fails
    }
  }

  async function calculatePoolStats(farm) {
    try {
      const provider = new ethers.providers.JsonRpcProvider('https://rpc.pulsechain.com');
      const lpToken = new ethers.Contract(farm.lpAddress, LP_TOKEN_ABI, provider);
      
      // Calculate TVL
      const totalLpInFarm = await lpToken.balanceOf(MASTERCHEF_ADDRESS);
      const totalSupply = await lpToken.totalSupply();
      const reserves = await lpToken.getReserves();
      
      let token0Price = 0;
      let token1Price = 0;
      
      if (farm.pair === 'LEAN-PLSX') {
        token0Price = TOKEN_PRICES.LEAN;
        token1Price = TOKEN_PRICES.PLSX;
      } else if (farm.pair === 'LIT-PLSX') {
        token0Price = TOKEN_PRICES.LIT;
        token1Price = TOKEN_PRICES.PLSX;
      } else if (farm.pair === 'LIT-LEAN') {
        token0Price = TOKEN_PRICES.LIT;
        token1Price = TOKEN_PRICES.LEAN;
      }

      const reserve0Value = ethers.utils.formatEther(reserves[0]) * token0Price;
      const reserve1Value = ethers.utils.formatEther(reserves[1]) * token1Price;
      const totalLpValue = reserve0Value + reserve1Value;

      const farmShare = totalLpInFarm.mul(ethers.BigNumber.from(1000000)).div(totalSupply).toNumber() / 1000000;
      const tvl = totalLpValue * farmShare;

      // Calculate APR
      let apr = 0;
      if (tvl > 0 && farmWeights[farm.pair]) {
        const yearlyRewards = BLOCKS_PER_YEAR * parseFloat(ethers.utils.formatEther(REWARD_PER_BLOCK));
        const poolYearlyRewards = yearlyRewards * (farmWeights[farm.pair] / 100);
        const rewardsValueUSD = poolYearlyRewards * TOKEN_PRICES.LIT;
        apr = (rewardsValueUSD / tvl) * 100;
      }

      // Update the farm object with new values
      farms = farms.map(f => {
        if (f.pair === farm.pair) {
          return { 
            ...f, 
            tvl, 
            apr,
            earned: userEarned[f.pair] ? parseFloat(userEarned[f.pair]) : 0
          };
        }
        return f;
      });
    } catch (error) {
      console.error(`Error calculating pool stats for ${farm.pair}:`, error);
    }
  }

  async function fetchPoolWeights() {
    try {
      const provider = new ethers.providers.JsonRpcProvider('https://rpc.pulsechain.com');
      const masterChef = new ethers.Contract(MASTERCHEF_ADDRESS, MASTERCHEF_ABI, provider);
      
      totalAllocPoint = await masterChef.totalAllocPoint();
      
      for (const farm of farms) {
        const poolInfo = await masterChef.poolInfo(farm.poolId);
        const allocPoint = poolInfo.allocPoint;
        const weight = (allocPoint.mul(1000).div(totalAllocPoint).toNumber()) / 10;
        farmWeights[farm.pair] = weight;
        
        // Calculate TVL and APR after getting the weight
        await calculatePoolStats(farm);
      }
    } catch (error) {
      console.error('Error fetching pool weights:', error);
    }
  }

  async function fetchUserEarned() {
    if (!wallet) return;

    try {
      const provider = new ethers.providers.Web3Provider(wallet.provider);
      const masterChef = new ethers.Contract(MASTERCHEF_ADDRESS, MASTERCHEF_ABI, provider);
      const address = await provider.getSigner().getAddress();

      for (const farm of farms) {
        const pending = await masterChef.pendingTokens(farm.poolId, address);
        userEarned[farm.pair] = ethers.utils.formatEther(pending);
      }

      // Update farms to trigger reactivity
      farms = farms.map(farm => ({
        ...farm,
        earned: userEarned[farm.pair] ? parseFloat(userEarned[farm.pair]) : 0
      }));
    } catch (error) {
      console.error('Error fetching user earned tokens:', error);
    }
  }

  // Refresh stats periodically
  let statsInterval: number;
  let earnedInterval: number;
    let priceInterval: number;

  onMount(() => {
    // Initial data fetch
    fetchTokenPrices().then(() => {
      fetchPoolWeights();
    });
    
    // Set up intervals
    statsInterval = setInterval(() => {
      fetchTokenPrices().then(() => {
        fetchPoolWeights();
      });
    }, 30000); // Refresh stats every 30 seconds
    
    priceInterval = setInterval(fetchTokenPrices, 300000); // Refresh prices every 5 minutes

    return () => {
      if (statsInterval) clearInterval(statsInterval);
      if (earnedInterval) clearInterval(earnedInterval);
      if (priceInterval) clearInterval(priceInterval);
    };
  });

  // Watch wallet changes to start/stop earned tokens refresh
  $: {
    if (wallet) {
      fetchUserEarned();
      if (!earnedInterval) {
        earnedInterval = setInterval(fetchUserEarned, 30000);
      }
    } else {
      if (earnedInterval) {
        clearInterval(earnedInterval);
        earnedInterval = undefined;
      }
      userEarned = {};
    }
  }

  onboard.state.select('wallets').subscribe((wallets) => {
    if (wallets.length > 0) {
      wallet = wallets[0];
    } else {
      wallet = null;
    }
  });
</script>

<Toaster />
<Navbar {wallet} {onboard} />

<main class="min-h-screen py-8 px-4 md:px-8">
  <div class="max-w-4xl mx-auto">
    <div class="text-center mb-12">
      <h1 class="text-4xl font-bold mb-4">Farms</h1>
      <div class="flex items-center justify-center gap-2 mb-2">
        <img 
          src="https://raw.githubusercontent.com/jaxnalia/lean-frontend-master/refs/heads/main/src/lib/images/lit.png" 
          alt="LIT" 
          class="w-6 h-6"
        />
        <p class="text-white font-semibold">Inflation: 10 LIT per block</p>
      </div>
      <p class="text-gray-400">Stake LP tokens to earn LIT rewards</p>
    </div>

    <div class="grid gap-4">
      {#each farms as farm (farm.pair)}
        <FarmCard 
          {...farm} 
          {wallet}
          weight={farmWeights[farm.pair] || 0}
        />
      {/each}
      
    </div>
  </div>
</main>