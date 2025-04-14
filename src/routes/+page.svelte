<!-- src/routes/+page.svelte -->
<script lang="ts">
	import { MapDisplay } from '$lib';
</script>

<svelte:head>
	<title>Kathmandu Shame Map</title>
	<meta name="description" content="Report pollution incidents in Kathmandu" />
</svelte:head>

<section class="container mx-auto px-4 py-8">
	<h1 class="mb-6 text-center text-3xl font-bold text-red-700">Kathmandu Shame Map (KSM)</h1>

	<p class="mb-6 text-center text-gray-700">
		Exposing Kathmandu's pollution problem. See reported incidents below. Report what you see!
	</p>

	<!-- Render the map component -->
	<div class="border border-gray-400 shadow-lg">
		<MapDisplay />
	</div>

	<!-- Other content will go here later, like the reporting form -->
	<div class="mt-8 text-center">
		<p>(Reporting form and filters coming soon...)</p>
	</div>
</section>
