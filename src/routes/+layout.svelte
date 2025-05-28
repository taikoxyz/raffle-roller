<script lang="ts">
	import '../app.css';

	let { children } = $props();
</script>

<svelte:head>
	<title>Taiko Raffle Roller</title>
</svelte:head>

{@render children()}
