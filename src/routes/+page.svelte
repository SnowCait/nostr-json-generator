<script lang="ts" context="module">
	interface Window {
		nostr: any;
	}
	declare const window: Window;
</script>

<script lang="ts">
	import { onMount } from 'svelte';
	import { createRxNostr, createRxOneshotReq, latest } from 'rx-nostr';

	let loggedIn = false;
	let browserExtensionNotFound = false;
	let pubkey = '';
	let relays: string[] = [];

	$: json = JSON.stringify(
		{
			names: {
				_: pubkey
			},
			relays: {
				[pubkey]: relays
			}
		},
		null,
		2
	);

	const loginStatusKey = 'nostr-json-generator:login';
	const defaultRelays = [
		'wss://nos.lol',
		'wss://relay.damus.io',
		'wss://relay-jp.nostr.wirednet.jp',
		'wss://nostr-relay.nokotaro.com',
		'wss://nostr.holybea.com',
		'wss://nostr.fediverse.jp',
		'wss://yabu.me'
	];

	async function login() {
		loggedIn = true;
		localStorage.setItem(loginStatusKey, JSON.stringify(loggedIn));

		await generate();
	}

	async function generate() {
		console.log('[generate]');
		pubkey = await window.nostr.getPublicKey();
		console.log('[pubkey]', pubkey);
		let nip07Relays: { [url: string]: { read: boolean; write: boolean } } = {};
		if (window.nostr.getRelays !== undefined) {
			nip07Relays = await window.nostr.getRelays();
			console.log('[NIP-07 relays]', nip07Relays);
		}

		const rxNostr = createRxNostr();
		await rxNostr.switchRelays([
			...[
				...Object.entries(nip07Relays)
					.filter(([, { write }]) => write)
					.map(([url]) => url)
			],
			...defaultRelays
		]);

		const rxReq = createRxOneshotReq({ filters: { authors: [pubkey], kinds: [10002] } });

		const observable = rxNostr.use(rxReq).pipe(latest());
		observable.subscribe((packet) => {
			console.log('[packet]', packet);

			const nip65Relays: string[][] = packet.event.tags;
			console.log('[NIP-65 relays]', nip65Relays);

			if (nip65Relays.length > 0) {
				relays = nip65Relays
					.filter(
						([tagName, , permission]) =>
							tagName === 'r' && (permission === undefined || permission === 'write')
					)
					.map(([, relay]) => relay);
			} else {
				relays = Object.entries(nip07Relays)
					.filter(([, { write }]) => write)
					.map(([relay]) => relay);
			}
		});
	}

	onMount(async () => {
		if (window.nostr === undefined) {
			browserExtensionNotFound = true;
			return;
		}

		const loginStatus = localStorage.getItem(loginStatusKey);
		console.log('[login]', loginStatus);
		if (loginStatus !== null && window.nostr !== undefined) {
			loggedIn = true;
			await generate();
		}
	});
</script>

<h1>nostr.json generator</h1>

{#if !loggedIn}
	<p>
		<a
			href="https://github.com/nostr-protocol/nips/blob/master/07.md"
			target="_blank"
			rel="noopener noreferrer">NIP-07</a
		> Browser Extension is required.
	</p>
	{#if browserExtensionNotFound}
		<p>Pelase install Browser Extension and reload.</p>
	{/if}
	<form on:submit|preventDefault={login}>
		<input type="submit" value="Generate" disabled={browserExtensionNotFound} />
	</form>
{/if}

<pre>{json}</pre>

<style>
	h1,
	p,
	form {
		text-align: center;
	}
</style>
