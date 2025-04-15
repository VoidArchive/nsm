<script lang="ts">
	import { browser } from '$app/environment';

	export type ReportData = {
		pollutionType: string;
		description: string;
		latitude: number;
		longitude: number;
		imageFile: File | null;
	};

	let { onSubmit }: { onSubmit: (data: ReportData) => Promise<void> | void } = $props();

	let pollutionType = $state<string>('Trash Dump');
	let description = $state<string>('');
	let latitude = $state<number | null>(null);
	let longitude = $state<number | null>(null);
	let imageFile = $state<File | null>(null);
	let imagePreviewUrl = $state<string | null>(null);
	let isSubmitting = $state<boolean>(false);
	let locationStatus = $state<string>('');
	let formError = $state<string | null>(null);

	const pollutionTypes = ['Trash Dump', 'Plastic Burning', 'Sewage Leak', 'Air Pollution', 'Other'];

	async function getCurrentLocation() {
		locationStatus = 'Getting location...';
		formError = null;

		if (!browser || !navigator.geolocation) {
			locationStatus = 'Geolocation is not supported by your browser.';
			return;
		}

		try {
			const position = await new Promise<GeolocationPosition>((resolve, reject) => {
				navigator.geolocation.getCurrentPosition(resolve, reject, {
					enableHighAccuracy: true,
					timeout: 10000,
					maximumAge: 0
				});
			});

			latitude = position.coords.latitude;
			longitude = position.coords.longitude;
			locationStatus = `Location: ${latitude.toFixed(5)}, ${longitude.toFixed(5)}`;
		} catch (err) {
			const error = err as GeolocationPositionError;
			switch (error.code) {
				case error.PERMISSION_DENIED:
					locationStatus = 'Permission denied. Please enable location access.';
					break;
				case error.POSITION_UNAVAILABLE:
					locationStatus = 'Location unavailable.';
					break;
				case error.TIMEOUT:
					locationStatus = 'Location request timed out.';
					break;
				default:
					locationStatus = 'Failed to get location.';
			}
			latitude = null;
			longitude = null;
		}
	}

	function handleFileSelect(event: Event) {
		const input = event.target as HTMLInputElement;
		const file = input.files?.[0];

		if (file && file.type.startsWith('image/')) {
			imageFile = file;
			formError = null;
		} else {
			formError = 'Please select a valid image file.';
			imageFile = null;
		}
	}

	async function handleSubmit() {
		formError = null;

		if (!pollutionType || !description.trim()) {
			formError = 'All fields marked with * are required.';
			return;
		}
		if (
			latitude === null ||
			longitude === null ||
			isNaN(latitude) ||
			isNaN(longitude) ||
			latitude < -90 ||
			latitude > 90 ||
			longitude < -180 ||
			longitude > 180
		) {
			formError = 'Please provide a valid location.';
			return;
		}

		isSubmitting = true;
		locationStatus = '';

		const reportData: ReportData = {
			pollutionType,
			description: description.trim(),
			latitude,
			longitude,
			imageFile
		};

		try {
			await onSubmit(reportData);
			console.log('Report submitted successfully.');
		} catch (error) {
			console.error('Submission failed:', error);
			formError = `Failed to submit: ${error instanceof Error ? error.message : 'Unknown error'}`;
		} finally {
			isSubmitting = false;
		}
	}

	$effect(() => {
		let currentUrl: string | null = null;

		if (imageFile) {
			currentUrl = URL.createObjectURL(imageFile);
			imagePreviewUrl = currentUrl;
		} else {
			imagePreviewUrl = null;
		}

		return () => {
			if (currentUrl) {
				URL.revokeObjectURL(currentUrl);
			}
		};
	});
</script>

<form
	on:submit|preventDefault={handleSubmit}
	class="space-y-4 rounded-lg border bg-white p-6 shadow-md"
>
	<h2 class="text-xl font-semibold text-gray-800">Report a Pollution Incident</h2>

	<!-- Pollution Type -->
	<div>
		<label class="block text-sm font-medium text-gray-700">Pollution Type*</label>
		<select bind:value={pollutionType} disabled={isSubmitting} class="...">
			{#each pollutionTypes as type}
				<option value={type}>{type}</option>
			{/each}
		</select>
	</div>

	<!-- Location -->
	<div class="space-y-2">
		<label class="block text-sm font-medium text-gray-700">Location*</label>
		<button type="button" on:click={getCurrentLocation} disabled={isSubmitting} class="...">
			Use Current Location
		</button>
		{#if locationStatus}
			<p class="text-sm text-gray-600">{locationStatus}</p>
		{/if}
		<div class="grid grid-cols-1 gap-4 md:grid-cols-2">
			<div>
				<label for="latitude">Latitude*</label>
				<input
					type="number"
					id="latitude"
					bind:value={latitude}
					step="any"
					required
					disabled={isSubmitting}
					class="..."
				/>
			</div>
			<div>
				<label for="longitude">Longitude*</label>
				<input
					type="number"
					id="longitude"
					bind:value={longitude}
					step="any"
					required
					disabled={isSubmitting}
					class="..."
				/>
			</div>
		</div>
	</div>

	<!-- Description -->
	<div>
		<label for="description">Description*</label>
		<textarea
			id="description"
			bind:value={description}
			rows={4}
			required
			disabled={isSubmitting}
			class="..."
		></textarea>
	</div>

	<!-- Image Upload -->
	<div>
		<label for="image">Upload Image</label>
		<input
			type="file"
			id="image"
			accept="image/*"
			on:change={handleFileSelect}
			disabled={isSubmitting}
			class="..."
		/>
		{#if imagePreviewUrl}
			<img src={imagePreviewUrl} alt="Preview" class="mt-2 max-h-40 rounded border" />
		{/if}
	</div>

	<!-- Error -->
	{#if formError}
		<p class="rounded-md bg-red-100 p-3 text-sm text-red-600">{formError}</p>
	{/if}

	<!-- Submit -->
	<div>
		<button type="submit" disabled={isSubmitting} class="...">
			{#if isSubmitting}
				Submitting...
			{:else}
				Submit Report
			{/if}
		</button>
	</div>
</form>
