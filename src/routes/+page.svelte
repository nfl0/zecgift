<script lang="ts">
	import { onMount } from 'svelte';
	import { browser } from '$app/environment';

	onMount(async () => {
		if (!browser) return;

		try {
			// Dynamically import to avoid SSR issues
			const { default: initWallet, initThreadPool, WebWallet, generate_seed_phrase } =
				await import('@chainsafe/webzjs-wallet');

			// Initialize WASM module
			await initWallet();

			// Initialize thread pool for parallel processing
			const numThreads = navigator.hardwareConcurrency || 4;
			await initThreadPool(numThreads);
			console.log(`Thread pool initialized with ${numThreads} threads`);

			// Generate random seed phrase
			const seedPhrase = generate_seed_phrase();
			console.log('Seed Phrase:', seedPhrase);

			// Create wallet connected to testnet via local gRPC-web proxy
			const wallet = new WebWallet('test', 'http://localhost:1234/testnet', 1);
			console.log('Wallet created, connecting to testnet...');

			// Get latest block height as birthday
			const latestBlock = await wallet.get_latest_block();
			const birthdayHeight = Number(latestBlock);
			console.log('Birthday Height (latest block):', birthdayHeight);

			// Create account with seed phrase
			const accountId = await wallet.create_account('default', seedPhrase, 0, birthdayHeight);
			console.log('Account created with ID:', accountId);

			// Get and log the address
			const address = await wallet.get_current_address(accountId);
			console.log('Zcash Address:', address);

			// Sync wallet
			console.log('Starting wallet sync...');
			await wallet.sync();
			console.log('Wallet has been synced');
		} catch (error) {
			console.error('Error:', error);
		}
	});
</script>

<h1>Hello World</h1>
