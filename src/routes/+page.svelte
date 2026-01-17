<script lang="ts">
	import { onMount, onDestroy } from 'svelte';
	import { browser } from '$app/environment';
	import { page } from '$app/stores';

	// State
	let address = $state('');
	let seedPhrase = $state('');
	let birthdayHeight = $state(0);
	let balance = $state(0); // in zatoshi
	let isLoading = $state(true);
	let isSyncing = $state(false);
	let copied = $state(false);
	let linkCopied = $state(false);
	let linkGenerated = $state(false);
	let isButtonHovered = $state(false);
	let wallet: any = null;
	let accountId: number = 0;
	let pollInterval: ReturnType<typeof setInterval> | null = null;

	// Convert zatoshi to ZEC
	function formatZec(zatoshi: number): string {
		if (zatoshi === 0) return '0';
		const zec = zatoshi / 100_000_000;
		// Remove trailing zeros but keep at least one decimal if needed
		return parseFloat(zec.toFixed(8)).toString();
	}

	// Truncate address for display
	function truncateAddress(addr: string): string {
		if (addr.length <= 20) return addr;
		return `${addr.slice(0, 12)}...${addr.slice(-8)}`;
	}

	// Truncate link for display
	function truncateLink(link: string): string {
		if (link.length <= 40) return link;
		return `${link.slice(0, 28)}...${link.slice(-8)}`;
	}

	// Reactive gift link
	let giftLink = $derived(
		browser && seedPhrase && birthdayHeight
			? `${window.location.origin}?gift=${btoa(JSON.stringify({ seed: seedPhrase, birthday: birthdayHeight }))}`
			: ''
	);

	// Copy to clipboard
	async function copyAddress() {
		await navigator.clipboard.writeText(address);
		copied = true;
		setTimeout(() => (copied = false), 2000);
	}

	// Generate the gift link (reveal it)
	function generateLink() {
		linkGenerated = true;
	}

	// Copy gift link to clipboard
	async function copyGiftLink() {
		await navigator.clipboard.writeText(giftLink);
		linkCopied = true;
		setTimeout(() => (linkCopied = false), 2000);
	}

	// Sync and update balance
	async function syncAndUpdateBalance() {
		if (!wallet || isSyncing) return;

		isSyncing = true;
		try {
			await wallet.sync();
			const summary = await wallet.get_wallet_summary();
			if (summary) {
				const balances = summary.account_balances;
				if (balances && balances.length > 0) {
					const [, bal] = balances[0];
					balance = (bal.sapling_balance || 0) + (bal.orchard_balance || 0) + (bal.unshielded_balance || 0);
				}
			}
		} catch (e) {
			console.error('Sync error:', e);
		} finally {
			isSyncing = false;
		}
	}

	// Parse gift data from URL
	function parseGiftFromUrl(): { seed: string; birthday: number } | null {
		if (!browser) return null;
		const giftParam = $page.url.searchParams.get('gift');
		if (!giftParam) return null;

		try {
			const decoded = atob(giftParam);
			return JSON.parse(decoded);
		} catch {
			return null;
		}
	}

	onMount(async () => {
		if (!browser) return;

		try {
			const { default: initWallet, initThreadPool, WebWallet, generate_seed_phrase } =
				await import('@chainsafe/webzjs-wallet');

			await initWallet();
			const numThreads = navigator.hardwareConcurrency || 4;
			await initThreadPool(numThreads);

			// Check if restoring from gift link
			const giftData = parseGiftFromUrl();

			wallet = new WebWallet('test', 'http://localhost:1234/testnet', 1);

			if (giftData) {
				// Restore from gift link
				seedPhrase = giftData.seed;
				birthdayHeight = giftData.birthday;
			} else {
				// Generate new ephemeral wallet
				seedPhrase = generate_seed_phrase();
				const latestBlock = await wallet.get_latest_block();
				birthdayHeight = Number(latestBlock);
			}

			accountId = await wallet.create_account('default', seedPhrase, 0, birthdayHeight);
			address = await wallet.get_current_address(accountId);

			isLoading = false;

			// Initial sync
			await syncAndUpdateBalance();

			// Poll every 10 seconds
			pollInterval = setInterval(syncAndUpdateBalance, 10000);
		} catch (error) {
			console.error('Error:', error);
			isLoading = false;
		}
	});

	onDestroy(() => {
		if (pollInterval) {
			clearInterval(pollInterval);
		}
	});
</script>

<div class="container">
	<div class="card">
		<div class="logo">ZecGift</div>

		<p class="tagline">
			Fund this address to create<br />a Zcash gift card
		</p>

		{#if isLoading}
			<div class="loading">
				<div class="spinner"></div>
				<p>Initializing wallet...</p>
			</div>
		{:else}
			<div class="address-container">
				<div class="address" title={address}>
					{truncateAddress(address)}
				</div>
				<button class="copy-btn" onclick={copyAddress} title="Copy address">
					{copied ? '✓' : '📋'}
				</button>
			</div>

			<div class="balance-container">
				<div class="balance">
					{formatZec(balance)} ZEC
					{#if isSyncing}
						<span class="balance-spinner"></span>
					{/if}
				</div>
			</div>

			<div class="gift-link-section">
				{#if !linkGenerated}
					<button
						class="generate-btn"
						onclick={generateLink}
						onmouseenter={() => (isButtonHovered = true)}
						onmouseleave={() => (isButtonHovered = false)}
					>
						Generate Gift Link
					</button>
					<p class="hint-text" class:highlighted={isButtonHovered}>
						Click generate link once account is funded
					</p>
				{:else}
					<div class="link-reveal">
						<div class="link-text" title={giftLink}>
							{giftLink ? truncateLink(giftLink) : 'Generating...'}
						</div>
						<button class="copy-link-btn" onclick={copyGiftLink} title="Copy gift link">
							{linkCopied ? 'Copied!' : 'Copy'}
						</button>
					</div>
					<p class="share-hint">Share this link with your recipient</p>
				{/if}
			</div>
		{/if}
	</div>
</div>

<style>
	:global(body) {
		margin: 0;
		padding: 0;
		font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, Oxygen, Ubuntu, Cantarell,
			sans-serif;
		background: linear-gradient(135deg, #f5f7fa 0%, #e4e8ec 100%);
		min-height: 100vh;
	}

	.container {
		min-height: 100vh;
		display: flex;
		align-items: center;
		justify-content: center;
		padding: 20px;
		box-sizing: border-box;
	}

	.card {
		background: white;
		border-radius: 24px;
		padding: 48px;
		box-shadow:
			0 4px 6px rgba(0, 0, 0, 0.05),
			0 10px 40px rgba(0, 0, 0, 0.08);
		max-width: 420px;
		width: 100%;
		text-align: center;
	}

	.logo {
		font-size: 32px;
		font-weight: 700;
		color: #1a1a1a;
		margin-bottom: 8px;
		letter-spacing: -0.5px;
	}

	.tagline {
		color: #666;
		font-size: 16px;
		line-height: 1.5;
		margin: 0 0 32px 0;
	}

	.loading {
		padding: 40px 0;
	}

	.spinner {
		width: 32px;
		height: 32px;
		border: 3px solid #e4e8ec;
		border-top-color: #1a1a1a;
		border-radius: 50%;
		animation: spin 0.8s linear infinite;
		margin: 0 auto 16px;
	}

	@keyframes spin {
		to {
			transform: rotate(360deg);
		}
	}

	.loading p {
		color: #666;
		margin: 0;
	}

	.address-container {
		display: flex;
		align-items: center;
		justify-content: center;
		gap: 12px;
		background: #f5f7fa;
		border-radius: 12px;
		padding: 16px 20px;
		margin-bottom: 24px;
	}

	.address {
		font-family: 'SF Mono', 'Fira Code', 'Consolas', monospace;
		font-size: 14px;
		color: #1a1a1a;
		word-break: break-all;
	}

	.copy-btn {
		background: none;
		border: none;
		font-size: 18px;
		cursor: pointer;
		padding: 4px;
		opacity: 0.6;
		transition: opacity 0.2s;
	}

	.copy-btn:hover {
		opacity: 1;
	}

	.balance-container {
		margin-bottom: 32px;
	}

	.balance {
		font-size: 36px;
		font-weight: 600;
		color: #1a1a1a;
		letter-spacing: -1px;
		display: flex;
		align-items: center;
		justify-content: center;
		gap: 12px;
	}

	.balance-spinner {
		width: 18px;
		height: 18px;
		border: 2px solid #e4e8ec;
		border-top-color: #1a1a1a;
		border-radius: 50%;
		animation: spin 0.8s linear infinite;
	}

	.gift-link-section {
		margin-top: 24px;
	}

	.generate-btn {
		background: #1a1a1a;
		color: white;
		border: none;
		border-radius: 12px;
		padding: 16px 32px;
		font-size: 16px;
		font-weight: 600;
		cursor: pointer;
		transition: all 0.2s;
		width: 100%;
	}

	.generate-btn:hover {
		background: #333;
		transform: translateY(-1px);
	}

	.generate-btn:active {
		transform: translateY(0);
	}

	.hint-text {
		font-size: 14px;
		color: #999;
		margin: 12px 0 0 0;
		transition: color 0.2s;
	}

	.hint-text.highlighted {
		color: #1a1a1a;
	}

	.link-reveal {
		display: flex;
		align-items: center;
		justify-content: space-between;
		gap: 12px;
		padding: 12px 0;
		animation: fadeSlideIn 0.3s ease-out;
	}

	@keyframes fadeSlideIn {
		from {
			opacity: 0;
			transform: translateY(-10px);
		}
		to {
			opacity: 1;
			transform: translateY(0);
		}
	}

	.link-text {
		font-family: 'SF Mono', 'Fira Code', 'Consolas', monospace;
		font-size: 13px;
		color: #1a1a1a;
		word-break: break-all;
		flex: 1;
		text-align: left;
	}

	.copy-link-btn {
		background: #1a1a1a;
		color: white;
		border: none;
		border-radius: 8px;
		padding: 8px 16px;
		font-size: 14px;
		font-weight: 500;
		cursor: pointer;
		transition: all 0.2s;
		white-space: nowrap;
	}

	.copy-link-btn:hover {
		background: #333;
	}

	.share-hint {
		font-size: 13px;
		color: #999;
		margin: 12px 0 0 0;
	}
</style>
