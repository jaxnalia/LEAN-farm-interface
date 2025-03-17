<script lang="ts">
  import { onMount } from 'svelte';
  import { ethers } from 'ethers';

  export let pair: string;
  export let apr: number;
  export let tvl: number;
  export let earned: number;
  export let staked: number;
  export let logo1: string;
  export let logo2: string;
  export let wallet: any;

  let isExpanded = false;
  let stakeAmount = '';
  let lpBalance = '0.00';
  let pendingRewards = '0.00';
  let userStaked = '0.00';
  let allowance = '0';
  let approving = false;
  
  const LP_TOKEN_ABI = [
    "function balanceOf(address owner) view returns (uint256)",
    "function decimals() view returns (uint8)",
    "function approve(address spender, uint256 amount) returns (bool)",
    "function allowance(address owner, address spender) view returns (uint256)"
  ];

  const MASTERCHEF_ADDRESS = '0xbE7f4fFfDe4241cA25eb27616aE3974aF0a023fD';
  
  const LP_TOKEN_ADDRESS = pair === 'LEAN-PLSX' 
    ? '0x9A5C8868a7B6c099238A87b7e4d0AE03f9CB4393' 
    : pair === 'LEAN-LIT'
    ? '0x567499ec3428c77F8B36bc6cA5221961330228f6'
    : undefined;

  const POOL_ID = pair === 'LEAN-PLSX' ? 0 : pair === 'LEAN-LIT' ? 2 : undefined;

  async function getLPBalance() {
    try {
      if (!wallet || !LP_TOKEN_ADDRESS) return;

      const provider = new ethers.providers.Web3Provider(wallet.provider);
      const signer = provider.getSigner();
      const address = await signer.getAddress();
      
      const lpToken = new ethers.Contract(LP_TOKEN_ADDRESS, LP_TOKEN_ABI, provider);
      const decimals = await lpToken.decimals();
      const balance = await lpToken.balanceOf(address);
      
      lpBalance = ethers.utils.formatUnits(balance, decimals);
    } catch (error) {
      console.error('Error fetching LP balance:', error);
    }
  }

  async function checkAllowance() {
    try {
      if (!wallet || !LP_TOKEN_ADDRESS) return;

      const provider = new ethers.providers.Web3Provider(wallet.provider);
      const signer = provider.getSigner();
      const address = await signer.getAddress();
      
      const lpToken = new ethers.Contract(LP_TOKEN_ADDRESS, LP_TOKEN_ABI, provider);
      const currentAllowance = await lpToken.allowance(address, MASTERCHEF_ADDRESS);
      
      allowance = ethers.utils.formatEther(currentAllowance);
    } catch (error) {
      console.error('Error checking allowance:', error);
    }
  }

  async function getUserInfo() {
    try {
      if (!wallet || POOL_ID === undefined) return;

      const provider = new ethers.providers.Web3Provider(wallet.provider);
      const signer = provider.getSigner();
      const address = await signer.getAddress();

      const masterChef = new ethers.Contract(MASTERCHEF_ADDRESS, MASTERCHEF_ABI, provider);
      
      // Get pending rewards
      const pending = await masterChef.pendingTokens(POOL_ID, address);
      pendingRewards = ethers.utils.formatEther(pending);

      // Get user staked amount
      const userInfo = await masterChef.userInfo(POOL_ID, address);
      userStaked = ethers.utils.formatEther(userInfo.amount);
      
    } catch (error) {
      console.error('Error fetching user info:', error);
    }
  }

  async function approve() {
    try {
      if (!wallet || !LP_TOKEN_ADDRESS || !stakeAmount) return;
      approving = true;

      const provider = new ethers.providers.Web3Provider(wallet.provider);
      const signer = provider.getSigner();
      
      const lpToken = new ethers.Contract(LP_TOKEN_ADDRESS, LP_TOKEN_ABI, signer);
      const amount = ethers.utils.parseEther(stakeAmount.toString());
      
      const tx = await lpToken.approve(MASTERCHEF_ADDRESS, amount);
      await tx.wait();
      
      await checkAllowance();
      approving = false;
      return true;
    } catch (error) {
      console.error('Error approving tokens:', error);
      approving = false;
      return false;
    }
  }

  async function handleStake() {
    try {
      if (!wallet || POOL_ID === undefined || !stakeAmount) return;

      const provider = new ethers.providers.Web3Provider(wallet.provider);
      const signer = provider.getSigner();
      
      const masterChef = new ethers.Contract(MASTERCHEF_ADDRESS, MASTERCHEF_ABI, signer);
      const amount = ethers.utils.parseEther(stakeAmount.toString());
      
      const tx = await masterChef.deposit(POOL_ID, amount);
      await tx.wait();
      
      // Refresh balances
      await getLPBalance();
      await getUserInfo();
      stakeAmount = '';
    } catch (error) {
      console.error('Error staking:', error);
    }
  }

  async function handleUnstake() {
    try {
      if (!wallet || POOL_ID === undefined || !stakeAmount) return;

      const provider = new ethers.providers.Web3Provider(wallet.provider);
      const signer = provider.getSigner();
      
      const masterChef = new ethers.Contract(MASTERCHEF_ADDRESS, MASTERCHEF_ABI, signer);
      const amount = ethers.utils.parseEther(stakeAmount.toString());
      
      const tx = await masterChef.withdraw(POOL_ID, amount);
      await tx.wait();
      
      // Refresh balances
      await getLPBalance();
      await getUserInfo();
      stakeAmount = '';
    } catch (error) {
      console.error('Error unstaking:', error);
    }
  }

  async function handleHarvest() {
    try {
      if (!wallet || POOL_ID === undefined) return;

      const provider = new ethers.providers.Web3Provider(wallet.provider);
      const signer = provider.getSigner();
      
      const masterChef = new ethers.Contract(MASTERCHEF_ADDRESS, MASTERCHEF_ABI, signer);
      
      // Deposit 0 amount to harvest rewards
      const tx = await masterChef.deposit(POOL_ID, 0);
      await tx.wait();
      
      // Refresh rewards
      await getUserInfo();
    } catch (error) {
      console.error('Error harvesting:', error);
    }
  }

  function setMaxBalance() {
    stakeAmount = lpBalance;
  }

  function toggleExpand() {
    isExpanded = !isExpanded;
    if (isExpanded && wallet) {
      getLPBalance();
      getUserInfo();
      checkAllowance();
    }
  }

  $: if (wallet) {
    if (isExpanded) {
      getLPBalance();
      getUserInfo();
      checkAllowance();
    }
  }

  $: needsApproval = wallet && stakeAmount && parseFloat(stakeAmount) > parseFloat(allowance);

  onMount(() => {
    if (wallet && isExpanded) {
      getLPBalance();
      getUserInfo();
      checkAllowance();
    }
  });

  const MASTERCHEF_ABI = [
    "function deposit(uint256 _pid, uint256 _amount)",
    "function withdraw(uint256 _pid, uint256 _amount)",
    "function pendingTokens(uint256 _pid, address _user) view returns (uint256)",
    "function userInfo(uint256, address) view returns (uint256 amount, uint256 rewardDebt)"
  ];
</script>

<div class="farm-card p-6 mb-4 cursor-pointer" on:click={toggleExpand}>
  <div class="flex justify-between items-center">
    <div class="flex items-center">
      <div class="relative w-12 h-12">
        <img src={logo1} alt="" class="absolute w-8 h-8 left-0" />
        <img src={logo2} alt="" class="absolute w-8 h-8 right-0 bottom-0" />
      </div>
      <div class="ml-4">
        <h3 class="text-xl font-bold">{pair}</h3>
        <p class="text-gray-400">Earn LIT</p>
      </div>
    </div>
    <div class="flex items-center gap-4">
      <div class="text-right">
        <p class="text-2xl font-bold text-[#ff3e00]">{apr}% APR</p>
        <p class="text-gray-400">TVL: ${tvl.toLocaleString()}</p>
      </div>
      <svg 
        class="w-6 h-6 text-gray-400 transition-transform duration-300 {isExpanded ? 'rotate-180' : ''}" 
        fill="none" 
        stroke="currentColor" 
        viewBox="0 0 24 24"
      >
        <path 
          stroke-linecap="round" 
          stroke-linejoin="round" 
          stroke-width="2" 
          d="M19 9l-7 7-7-7"
        />
      </svg>
    </div>
  </div>

  {#if isExpanded}
    <div 
      class="mt-6 border-t border-gray-700 pt-4"
      on:click|stopPropagation
    >
      <div class="grid grid-cols-2 gap-4 mb-4">
        <div>
          <p class="text-gray-400">Earned</p>
          <p class="text-xl font-bold">{parseFloat(pendingRewards).toFixed(4)} LIT</p>
          <button 
            class="btn-primary mt-2 w-full" 
            disabled={!wallet}
            on:click={handleHarvest}
          >
            Harvest
          </button>
        </div>
        <div>
          <p class="text-gray-400">Staked</p>
          <p class="text-xl font-bold">{parseFloat(userStaked).toFixed(4)} LP</p>
        </div>
      </div>

      <div class="mt-4">
        <div class="relative">
          <input 
            type="number" 
            bind:value={stakeAmount}
            placeholder="Enter amount to stake"
            class="input-amount w-full mb-2 pr-32"
            disabled={!wallet}
          />
          {#if wallet}
            <button 
              class="absolute right-2 top-0 h-full flex items-center text-white hover:text-[#ff3e00] transition-colors"
              on:click={setMaxBalance}
            >
              Balance: {parseFloat(lpBalance).toFixed(4)}
            </button>
          {/if}
          
        </div>
        <div class="grid grid-cols-2 gap-2">
          {#if needsApproval}
            <button 
              class="btn-primary col-span-2" 
              on:click={approve}
              disabled={approving}
            >
              {approving ? 'Approving...' : 'Approve LP Token'}
            </button>
          {:else}
            <button 
              class="btn-primary" 
              on:click={handleStake}
              disabled={!wallet || !stakeAmount || parseFloat(stakeAmount) <= 0}
            >
              Stake
            </button>
            <button 
              class="bg-[#2a2d3a] text-white font-semibold py-2 px-4 rounded-lg hover:bg-[#353849]"
              on:click={handleUnstake}
              disabled={!wallet || !stakeAmount || parseFloat(stakeAmount) <= 0}
            >
              Unstake
            </button>
          {/if}
          
        </div>
      </div>
    </div>
  {/if}
  
</div>