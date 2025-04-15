<!-- src/lib/components/MapDisplay.svelte -->
<script lang="ts">
	import { onMount } from 'svelte';
	import { browser } from '$app/environment';
	// Use L_Type for type annotations to avoid ambiguity with the runtime 'L' variable
	import type L_Type from 'leaflet';
	import type { SupabaseClient } from '@supabase/supabase-js';

	// --- Props ---
	export let supabase: SupabaseClient;

	// --- Component State ---
	let mapContainer: HTMLDivElement | null = null; // Element to bind the map to
	let mapInstance: L_Type.Map | null = null; // Holds the Leaflet Map instance
	let markersLayer: L_Type.LayerGroup | null = null; // Holds the report markers
	let isLoading: boolean = true; // Tracks loading state (map init + initial data fetch)
	let mapError: string | null = null; // Stores error messages

	// Variable to hold the Leaflet library instance once loaded dynamically
	// Declared at component scope so it's accessible by multiple functions
	let L: typeof L_Type | null = null;

	// --- Types for Fetched Data ---
	type PollutionReport = {
		id: string; // Assuming UUID
		created_at: string;
		latitude: number;
		longitude: number;
		pollution_type: string;
		description: string;
		image_url: string | null;
	};
	let reports: PollutionReport[] = []; // Array to store fetched reports

	// --- Map Constants ---
	const initialCoords: L_Type.LatLngTuple = [27.7172, 85.324];
	const initialZoom = 13;

	// --- Fetch Reports Function ---
	async function fetchReports() {
		// Don't reset isLoading here, it's handled by the initial load cycle
		// isLoading = true; // Avoid resetting isLoading on subsequent refreshes
		mapError = null; // Clear previous errors before fetching
		console.log('[MapDisplay] Fetching pollution reports...');

		// Guard clause: Ensure Supabase client is available
		if (!supabase) {
			console.error('[MapDisplay] Supabase client not available for fetching reports.');
			mapError = 'Backend connection error.';
			isLoading = false; // Stop loading if no client
			return;
		}

		try {
			const { data, error } = await supabase
				.from('pollution_reports')
				.select('id, created_at, latitude, longitude, pollution_type, description, image_url')
				// .eq('is_approved', true) // Optional: Filter for approved later
				.order('created_at', { ascending: false })
				.limit(500); // Sensible limit

			if (error) {
				throw error; // Throw to be caught below
			}

			reports = data || [];
			console.log(`[MapDisplay] Fetched ${reports.length} reports.`);
			updateMarkers(); // Update markers on the map with new data
		} catch (err: any) {
			console.error('[MapDisplay] Error fetching reports:', err);
			mapError = `Failed to load reports: ${err.message}`;
			reports = []; // Clear data on error
			updateMarkers(); // Clear markers on error
		} finally {
			// Only set isLoading to false after the *initial* fetch completes
			// Subsequent fetches (refreshes) shouldn't show the main loading overlay
			// We rely on the onMount flow to eventually set isLoading = false
		}
	}

	// --- Update Markers Function ---
	function updateMarkers() {
		// Guard clauses: Ensure map, marker layer, and Leaflet library are ready
		if (!mapInstance) {
			console.warn('[MapDisplay] updateMarkers called before mapInstance is ready.');
			return;
		}
		if (!markersLayer) {
			console.warn('[MapDisplay] updateMarkers called before markersLayer is ready.');
			return;
		}
		if (!L) {
			console.warn('[MapDisplay] updateMarkers called before Leaflet (L) is loaded.');
			return;
		}

		console.log('[MapDisplay] Updating markers...');
		markersLayer.clearLayers(); // Clear previous markers

		reports.forEach((report) => {
			// Skip reports with invalid coordinates
			if (
				report.latitude == null ||
				report.longitude == null ||
				isNaN(report.latitude) ||
				isNaN(report.longitude)
			) {
				console.warn(`[MapDisplay] Skipping report ID ${report.id} due to invalid coordinates.`);
				return;
			}

			try {
				// Use the component-scoped 'L' to create the marker
				const marker = L!.marker([report.latitude, report.longitude]);

				// Create popup content HTML string
				let popupContent = `
                    <div class="ksm-popup space-y-1 max-w-xs" style="max-height: 250px; overflow-y: auto;">
                        <strong class="text-base capitalize">${report.pollution_type?.replace(/_/g, ' ') || 'Report'}</strong>
                        <p class="text-sm text-gray-700">${report.description || 'No description provided.'}</p>
                        ${
													report.image_url
														? `<a href="${report.image_url}" target="_blank" rel="noopener noreferrer" class="block mt-2"><img src="${report.image_url}" alt="Pollution report image" class="max-h-32 w-auto rounded border border-gray-300" loading="lazy"></a>`
														: ''
												}
                        <p class="text-xs text-gray-500 pt-1">Reported: ${new Date(report.created_at).toLocaleString()}</p>
                        <!-- Example: Add Share Link (Replace with actual logic) -->
                        <!-- <a href="/report/${report.id}" target="_blank" class="text-xs text-blue-600 hover:underline">Details/Share</a> -->
                    </div>
                `;

				marker.bindPopup(popupContent);
				markersLayer!.addLayer(marker); // Add marker to the layer group
			} catch (error) {
				console.error(`[MapDisplay] Error creating marker for report ID ${report.id}:`, error);
			}
		});

		// Ensure the layer group is on the map (should be added during init)
		if (!mapInstance.hasLayer(markersLayer)) {
			console.warn('[MapDisplay] markersLayer was not found on map, adding it again.');
			markersLayer.addTo(mapInstance);
		}
		console.log(`[MapDisplay] Finished updating markers. ${reports.length} reports processed.`);
	}

	// --- Lifecycle: Initialization and Cleanup ---
	onMount(() => {
		console.log('[MapDisplay] Component attempting to mount...');
		// Ensure cleanup function is always returned, even if init fails early
		let isMounted = true;

		const initializeMap = async () => {
			// Only run map initialization logic in the browser
			if (!browser) {
				console.log('[MapDisplay] Not in browser, skipping map initialization.');
				isLoading = false; // Ensure loading state is false if SSR
				return;
			}

			// Ensure we only initialize once, even with HMR
			if (mapInstance) {
				console.log('[MapDisplay] Map already initialized, skipping re-initialization.');
				isLoading = false; // Already loaded
				return;
			}

			// Check if the container div is ready
			if (!mapContainer) {
				console.error('[MapDisplay] Map container div not found on mount!');
				mapError = 'Map container failed to load.';
				isLoading = false;
				return;
			}

			console.log('[MapDisplay] Starting map initialization...');
			try {
				// Dynamically import Leaflet library
				console.log('[MapDisplay] Importing Leaflet...');
				const L_imported = (await import('leaflet')).default;
				// Assign to component-scoped L
				L = L_imported;
				console.log('[MapDisplay] Leaflet imported successfully.', L);

				// Double-check L was loaded
				if (!L) {
					throw new Error('Leaflet library (L) failed to load or is null.');
				}

				// Initialize the layer group *after* L is loaded
				console.log('[MapDisplay] Initializing markersLayer...');
				markersLayer = L.layerGroup();
				console.log('[MapDisplay] markersLayer initialized.', markersLayer);

				// Initialize the map instance
				console.log('[MapDisplay] Initializing map instance...');
				mapInstance = L.map(mapContainer).setView(initialCoords, initialZoom);
				console.log('[MapDisplay] Map instance initialized.', mapInstance);

				// Add the base tile layer
				L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
					attribution:
						'© <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors',
					maxZoom: 19
				}).addTo(mapInstance);
				console.log('[MapDisplay] Tile layer added.');

				// Add the markers layer group to the map
				markersLayer.addTo(mapInstance);
				console.log('[MapDisplay] Markers layer added to map.');

				// Fetch initial reports now that map is ready
				await fetchReports();
			} catch (err: any) {
				console.error('[MapDisplay] Initialization failed:', err);
				mapError = `Map initialization failed: ${err.message}`;
				// Ensure L, mapInstance, markersLayer are null if init failed partway
				L = null;
				mapInstance = null;
				markersLayer = null;
			} finally {
				// This ensures isLoading becomes false once init attempt is complete
				console.log('[MapDisplay] Initialization attempt finished.');
				isLoading = false;
			}
		};

		initializeMap(); // Start the async initialization

		// Cleanup function: Runs when the component is unmounted
		return () => {
			isMounted = false;
			console.log('[MapDisplay] Component unmounting...');
			if (mapInstance) {
				console.log('[MapDisplay] Removing map instance.');
				mapInstance.remove();
				mapInstance = null;
			}
			// Explicitly nullify to help GC and prevent potential stale references
			L = null;
			markersLayer = null;
			mapContainer = null; // Should be handled by Svelte's binding, but doesn't hurt
			reports = []; // Clear data
		};
	});

	// --- Public function to allow parent to trigger refresh ---
	// Called from +page.svelte after successful submission
	export async function refreshData() {
		if (!mapInstance) {
			console.warn('[MapDisplay] refreshData called, but map is not ready.');
			return; // Don't try fetching if map isn't there
		}
		console.log('[MapDisplay] Refreshing map data via refreshData()...');
		isLoading = true; // Indicate loading during refresh
		mapError = null;
		try {
			await fetchReports();
		} finally {
			isLoading = false; // Turn off loading indicator after refresh
		}
	}
</script>

<!-- Template Structure -->
<div class="relative h-full w-full">
	<!-- Map container div - ensure it has a height defined by parent or CSS -->
	<div
		bind:this={mapContainer}
		class="z-0 h-full w-full"
		role="application"
		aria-label="Pollution Map"
	>
		<!-- Fallback content if needed, though map init handles loading/error -->
	</div>

	<!-- Loading Overlay - Covers the map area during initial load or refresh -->
	{#if isLoading}
		<div
			class="bg-opacity-75 pointer-events-none absolute inset-0 z-10 flex items-center justify-center bg-white"
		>
			<div class="text-center">
				<svg
					class="mx-auto mb-2 h-8 w-8 animate-spin text-blue-600"
					xmlns="http://www.w3.org/2000/svg"
					fill="none"
					viewBox="0 0 24 24"
				>
					<circle class="opacity-25" cx="12" cy="12" r="10" stroke="currentColor" stroke-width="4"
					></circle>
					<path
						class="opacity-75"
						fill="currentColor"
						d="M4 12a8 8 0 018-8V0C5.373 0 0 5.373 0 12h4zm2 5.291A7.962 7.962 0 014 12H0c0 3.042 1.135 5.824 3 7.938l3-2.647z"
					></path>
				</svg>
				<p class="text-lg font-semibold text-gray-700">Loading Reports...</p>
			</div>
		</div>
	{/if}

	<!-- Error Message Display - Shows map-related errors -->
	{#if mapError && !isLoading}
		<div
			class="absolute top-0 right-0 left-0 z-10 border-b border-red-200 bg-red-100 p-2 text-center text-sm text-red-800"
		>
			<strong>Map Error:</strong>
			{mapError}
		</div>
	{/if}
</div>

<!-- Component Styles -->
<style>
	/* Ensure Leaflet's CSS works well (can be global if needed) */
	:global(.leaflet-container) {
		line-height: normal; /* Reset potentially conflicting base styles */
		background-color: #eee; /* Basic background while tiles load */
	}
	:global(.leaflet-popup-content-wrapper, .leaflet-popup-tip) {
		background-color: white;
		color: #333;
		box-shadow: 0 3px 14px rgba(0, 0, 0, 0.4);
	}
	/* Style for the custom popup content div */
	:global(.ksm-popup) {
		font-family:
			system-ui,
			-apple-system,
			BlinkMacSystemFont,
			'Segoe UI',
			Roboto,
			Oxygen,
			Ubuntu,
			Cantarell,
			'Open Sans',
			'Helvetica Neue',
			sans-serif;
		font-size: 14px;
	}
	:global(.ksm-popup img) {
		max-width: 100%; /* Allow image to use popup width */
		max-height: 150px; /* But limit height */
		height: auto;
		margin-top: 4px;
		display: block; /* Prevents extra space below image */
	}
</style>
