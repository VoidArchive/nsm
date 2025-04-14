<script lang="ts">
	import { onMount } from 'svelte';
	import { browser } from '$app/environment';
	import type L from 'leaflet';

	let mapContainer: HTMLDivElement | null = null;
	let mapInstance: L.Map | null = null;

	// Kathmandu coord
	const initialCoords: L.LatLngTuple = [27.7172, 85.324];
	const initialZoom = 13;

	onMount(() => {
		const initializeMap = async () => {
			if (browser) {
				// Dynamic import
				const L = (await import('leaflet')).default;

				if (mapContainer && !mapInstance) {
					mapInstance = L.map(mapContainer, {
						attributionControl: false
					}).setView(initialCoords, initialZoom);

					L.control.attribution({ prefix: false }).addTo(mapInstance);

					L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
						attribution:
							'© <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
					}).addTo(mapInstance);

					console.log('Leaflet map initialized');
				}
			}
		};

		initializeMap().catch((error) => {
			console.error('Failed to initialize map');
		});

		return () => {
			if (mapInstance) {
				console.log('Destroying map instance');
				mapInstance.remove();
				mapInstance = null;
			}
		};
	});

	export function getMapInstance(): L.Map | null {
		return mapInstance;
	}
</script>

<div
	bind:this={mapContainer}
	class="z-0 h-[600px] w-full"
	role="application"
	aria-label="Pollution Map"
>
	<p>Loading map...</p>
</div>

<style>
	:global(.leaflet-container) {
		line-height: normal;
	}

	:global(.leaflet-popup-content-wrapper, .leaflet-popup-tip) {
		background-color: white; /* Example */
		color: #333; /* Example */
		box-shadow: 0 3px 14px rgba(0, 0, 0, 0.4); /* Example */
	}
</style>
