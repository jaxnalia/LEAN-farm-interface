<script lang="ts">
  import { onMount } from 'svelte';
  import { ethers } from 'ethers';
  import toast from 'svelte-french-toast';

  export let pair: string;
  export let apr: number;
  export let tvl: number;
  export let earned: number;
  export let staked: number;
  export let logo1: string;
  export let logo2: string;
  export let wallet: any;
  export let weight: number;
  export let poolId: number;
  export let lpAddress: string;

  let isExpanded = false;
  let stakeAmount = '';
  let unstakeAmount = '';
  let lpBalance = '0.00';
  let userStaked = '0.00';
  let allowance = '0';
  let approving = false;
  let harvesting = false;
  
  const LP_TOKEN_ABI = [
    "function balanceOf(address owner) view returns (uint256)",
    "function decimals() view returns (uint8)",
    "function approve(address spender, uint256 amount) returns (bool)",
    "function allowance(address owner, address spender) view returns (uint256)"
  ];

  const MASTERCHEF_ADDRESS = '0xbE7f4fFfDe4241cA25eb27616aE3974aF0a023fD';
  const MASTERCHEF_ABI = [
    "function deposit(uint256 _pid, uint256 _amount)",
    "function withdraw(uint256 _pid, uint256 _amount)",
    "function userInfo(uint256, address) view returns (uint256 amount, uint256 rewardDebt)"
  ];

  function handleError(error: any, toastId?: string) {
    console.error('Transaction error:', error);
    
    if (error.code === 4001 || 
        error.code === 'ACTION_REJECTED' || 
        error.message?.includes('User denied') || 
        error.message?.includes('User rejected')) {
      toast.error('Transaction was rejected', { id: toastId });
    } else {
      toast.error('Transaction failed', { id: toastId });
    }
  }

  async function getLPBalance() {
    try {
      if (!wallet || !lpAddress) return;

      const provider = new ethers.providers.Web3Provider(wallet.provider);
      const signer = provider.getSigner();
      const address = await signer.getAddress();
      
      const lpToken = new ethers.Contract(lpAddress, LP_TOKEN_ABI, provider);
      const decimals = await lpToken.decimals();
      const balance = await lpToken.balanceOf(address);
      
      lpBalance = ethers.utils.formatUnits(balance, decimals);
    } catch (error) {
      console.error('Error fetching LP balance:', error);
      toast.error('Failed to fetch LP balance');
    }
  }

  async function checkAllowance() {
    try {
      if (!wallet || !lpAddress) return;

      const provider = new ethers.providers.Web3Provider(wallet.provider);
      const signer = provider.getSigner();
      const address = await signer.getAddress();
      
      const lpToken = new ethers.Contract(lpAddress, LP_TOKEN_ABI, provider);
      const currentAllowance = await lpToken.allowance(address, MASTERCHEF_ADDRESS);
      
      allowance = ethers.utils.formatEther(currentAllowance);
    } catch (error) {
      console.error('Error checking allowance:', error);
      toast.error('Failed to check token allowance');
    }
  }

  async function getUserInfo() {
    try {
      if (!wallet || poolId === undefined) return;

      const provider = new ethers.providers.Web3Provider(wallet.provider);
      const signer = provider.getSigner();
      const address = await signer.getAddress();

      const masterChef = new ethers.Contract(MASTERCHEF_ADDRESS, MASTERCHEF_ABI, provider);
      const userInfo = await masterChef.userInfo(poolId, address);
      userStaked = ethers.utils.formatEther(userInfo.amount);
    } catch (error) {
      console.error('Error fetching user info:', error);
      toast.error('Failed to fetch user information');
    }
  }

  async function approve() {
    const toastId = toast.loading('Approving tokens...');
    try {
      if (!wallet || !lpAddress || !stakeAmount) return;
      approving = true;

      const provider = new ethers.providers.Web3Provider(wallet.provider);
      const signer = provider.getSigner();
      
      const lpToken = new ethers.Contract(lpAddress, LP_TOKEN_ABI, signer);
      
      // Approve a large amount (max uint256) for convenience, or the specific amount
      // Using max approval to avoid needing to approve again for future transactions
      const maxAmount = ethers.constants.MaxUint256;
      
      const tx = await lpToken.approve(MASTERCHEF_ADDRESS, maxAmount);
      await tx.wait();
      
      await checkAllowance();
      approving = false;
      toast.success('Tokens approved successfully', { id: toastId });
      await handleStake(); // Automatically stake after approval
      return true;
    } catch (error) {
      approving = false;
      handleError(error, toastId);
      return false;
    }
  }

  async function handleStake() {
    const toastId = toast.loading('Staking tokens...');
    try {
      if (!wallet || poolId === undefined || !stakeAmount) return;

      const provider = new ethers.providers.Web3Provider(wallet.provider);
      const signer = provider.getSigner();
      
      const masterChef = new ethers.Contract(MASTERCHEF_ADDRESS, MASTERCHEF_ABI, signer);
      const amount = ethers.utils.parseEther(stakeAmount);
      
      const tx = await masterChef.deposit(poolId, amount);
      await tx.wait();
      
      await getLPBalance();
      await getUserInfo();
      stakeAmount = '';
      toast.success('Tokens staked successfully', { id: toastId });
    } catch (error) {
      handleError(error, toastId);
    }
  }

  async function handleUnstake() {
    const toastId = toast.loading('Unstaking tokens...');
    try {
      if (!wallet || poolId === undefined || !unstakeAmount) return;

      const provider = new ethers.providers.Web3Provider(wallet.provider);
      const signer = provider.getSigner();
      
      const masterChef = new ethers.Contract(MASTERCHEF_ADDRESS, MASTERCHEF_ABI, signer);
      const amount = ethers.utils.parseEther(unstakeAmount);
      
      const tx = await masterChef.withdraw(poolId, amount);
      await tx.wait();
      
      await getLPBalance();
      await getUserInfo();
      unstakeAmount = '';
      toast.success('Tokens unstaked successfully', { id: toastId });
    } catch (error) {
      handleError(error, toastId);
    }
  }

  async function handleHarvest() {
    const toastId = toast.loading('Harvesting rewards...');
    try {
      if (!wallet || poolId === undefined) return;
      harvesting = true;
      const provider = new ethers.providers.Web3Provider(wallet.provider);
      const signer = provider.getSigner();
      
      const masterChef = new ethers.Contract(MASTERCHEF_ADDRESS, MASTERCHEF_ABI, signer);
      
      const tx = await masterChef.deposit(poolId, 0);
      await tx.wait();
      harvesting = false;
      toast.success('Rewards harvested successfully', { id: toastId });
    } catch (error) {
      harvesting = false;
      handleError(error, toastId);
    }
  }

  function setMaxStakeBalance() {
    stakeAmount = lpBalance;
  }

  function setMaxUnstakeBalance() {
    unstakeAmount = userStaked;
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

  // Check if approval is needed - compare the stake amount with current allowance
  $: needsApproval = wallet && stakeAmount && parseFloat(stakeAmount) > 0 && parseFloat(stakeAmount) > parseFloat(allowance);

  onMount(() => {
    if (wallet && isExpanded) {
      getLPBalance();
      getUserInfo();
      checkAllowance();
    }
  });
</script>

<!-- svelte-ignore a11y-no-static-element-interactions -->
<!-- svelte-ignore a11y-click-events-have-key-events -->
<div class="farm-card p-6 mb-4 cursor-pointer" on:click={toggleExpand}>
  <div class="flex justify-between items-center">
    <div class="flex items-center">
      <div class="relative w-12 h-12">
        <img src={logo1} alt="" class="absolute w-8 h-8 left-0" />
        <img src={logo2} alt="" class="absolute w-8 h-8 right-0 bottom-0" />
      </div>
      <div class="ml-4">
        <h3 class="text-lg font-bold">{pair}</h3>
        <p class="text-gray-400">Earn LIT</p>
      </div>
    </div>
    <div class="flex items-center gap-4">
      <div class="text-right">
        <div class="flex items-center gap-2 mb-1">
          <span class="text-yellow-400 border border-yellow-400 px-1 rounded-md font-semibold">{weight}x</span>
          <span class="text-xl font-bold text-[#ffffff]">{apr.toFixed(2)}% APY</span>
        </div>
        <p class="text-gray-400">TVL: ${tvl.toLocaleString(undefined, { maximumFractionDigits: 2 })}</p>
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
          <p class="text-xl font-bold">{earned.toFixed(4)} LIT</p>
          <button 
            class="btn-primary mt-2 w-full" 
            disabled={harvesting}
            on:click={handleHarvest}
          >
            {harvesting ? 'Harvesting...' : 'Harvest'}
          </button>
        </div>
        <div>
          <p class="text-gray-400">Staked</p>
          <p class="text-xl font-bold">{parseFloat(userStaked).toFixed(4)} LP</p>
        </div>
      </div>

      <div class="mt-4 grid gap-4">
        <div>
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
                class="absolute right-2 top-0 h-full flex items-center text-white hover:underline transition-colors"
                on:click={setMaxStakeBalance}
              >
                Balance: {parseFloat(lpBalance).toFixed(4)}
              </button>
            {/if}
          </div>
          {#if needsApproval}
            <button 
              class="btn-primary w-full" 
              on:click={approve}
              disabled={approving}
            >
              {approving ? 'Approving...' : 'Approve LP Token'}
            </button>
          {:else}
            <button 
              class="btn-primary w-full" 
              on:click={handleStake}
              disabled={!wallet || !stakeAmount || parseFloat(stakeAmount) <= 0}
            >
              Stake
            </button>
          {/if}
        </div>

        <div>
          <div class="relative">
            <input 
              type="number" 
              bind:value={unstakeAmount}
              placeholder="Enter amount to unstake"
              class="input-amount w-full mb-2 pr-32"
              disabled={!wallet}
            />
            {#if wallet}
              <button 
                class="absolute right-2 top-0 h-full flex items-center text-white hover:underline transition-colors"
                on:click={setMaxUnstakeBalance}
              >
                Staked: {parseFloat(userStaked).toFixed(4)}
              </button>
            {/if}
          </div>
          <button 
            class="bg-[#2a2d3a] transition-all duration-300 text-white font-semibold py-2 px-4 rounded-lg hover:bg-[#353849] w-full"
            on:click={handleUnstake}
            disabled={!wallet || !unstakeAmount || parseFloat(unstakeAmount) <= 0}
          >
            Unstake
          </button>
        </div>
      </div>
    </div>
  {/if}
</div>