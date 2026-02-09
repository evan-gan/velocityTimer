<script lang="ts">
	let { children } = $props();
</script>

<svelte:head>
	<link
		href="https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;700;900&display=swap"
		rel="stylesheet"
	/>
	<title>Shot Velocity Timer</title>
</svelte:head>

{@render children()}

<style>
	:global(*) {
		margin: 0;
		padding: 0;
		box-sizing: border-box;
	}

	:global(body) {
		font-family: 'DM Sans', sans-serif;
		background: #f5f5f5;
		color: #000;
		min-height: 100vh;
	}
</style>
