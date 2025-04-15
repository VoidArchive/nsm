<!-- src/routes/+page.svelte -->
<script lang="ts">
	import { MapDisplay } from '$lib';
	import { ReportForm } from '$lib';
	import type { ReportData } from '$lib';
	import { browser } from '$app/environment';
	import { onMount } from 'svelte';
	import { createClient, type SupabaseClient } from '@supabase/supabase-js';

	let submissionMessage: string | null = null;
	let isProcessing: boolean = false;

	const SUPABASE_URL = import.meta.env.VITE_SUPABASE_URL;
	const SUPABASE_ANON_KEY = import.meta.env.VITE_SUPABASE_ANON_KEY;

	let supabase: SupabaseClient | null = null;

	let mapDisplayComponent: MapDisplay | null = null;

	onMount(() => {
		if (!SUPABASE_URL || !SUPABASE_ANON_KEY) {
			console.error('Supabase URL or Anon Key is missing. Check .env file');
			submissionMessage = 'Configuration error: Cannot connect to backend';
			return;
		}
		if (browser) {
			supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
			console.log('Supabase client initialized');
		}
	});

	async function handleReportSubmit(reportDetails: ReportData) {
		isProcessing = true;
		submissionMessage = 'Processing report...';
		console.log('Received report data in parent:', reportDetails);

		if (!supabase) {
			submissionMessage = 'Error: Backend connection not ready. Please refresh';
			isProcessing = false;
			console.error('Supabase client not initialized');
			return;
		}

		let uploadedImageUrl: string | null = null;
		try {
			if (reportDetails.imageFile) {
				submissionMessage = 'Uploading image';
				const file = reportDetails.imageFile;
				const fileExt = file.name.split('.').pop();
				const randomSuffix = Math.random().toString(36).substring(2, 8);
				const filePath = `public/${Date.now()}_${randomSuffix}.${fileExt}`;
				const { data: uploadData, error: uploadError } = await supabase.storage
					.from('pollution-images')
					.upload(filePath, file);

				if (uploadError) {
					console.error('Image Upload Error:', uploadError);
					throw new Error(`Image upload failed: ${uploadError.message}`);
				}

				const { data: urlData } = supabase.storage.from('pollution-images').getPublicUrl(filePath);

				if (!urlData || !urlData.publicUrl) {
					console.warn('Could not get public URL for uploaded image, but upload succeeded');
					uploadedImageUrl = null;
				} else {
					uploadedImageUrl = urlData.publicUrl;
					console.log('Image uploaded:', uploadedImageUrl);
				}

				submissionMessage = 'Saving report details...';

				if (reportDetails.latitude === null || reportDetails.longitude === null) {
					throw new Error('Latitude or Longitude is missing');
				}

				const reportToInsert = {
					latitude: reportDetails.latitude,
					longitude: reportDetails.longitude,
					pollution_type: reportDetails.pollutionType,
					description: reportDetails.description,
					image_url: uploadedImageUrl
				};

				const { error: insertError } = await supabase
					.from('pollution_reports')
					.insert(reportToInsert);

				if (insertError) {
					console.error('Database Insert error:', insertError);
					throw new Error(`Failed to save report: ${insertError.message}`);
				}
				submissionMessage = 'Report submitted successfully!';
				console.log('Report successfully inserted into database');
			}

			if (mapDisplayComponent) {
				mapDisplayComponent.refreshData();
			} else {
				console.warn('Could not refresh map - component reference not available.');
			}
		} catch (error: any) {
			console.error('Submission failed: ', error);
			submissionMessage = `Error: ${error.message || 'An unknown error occurred during submission'}`;
		} finally {
			isProcessing = false;
			setTimeout(() => {
				submissionMessage = null;
			}, 7000);
		}
	}
</script>

<svelte:head>
	<title>Kathmandu Shame Map</title>
	<meta name="description" content="Report pollution incidents in Kathmandu" />
</svelte:head>

<section class="container mx-auto space-y-8 px-4 py-8">
	<div>
		<h1 class="mb-4 text-center text-3xl font-bold text-red-700">Kathmandu Shame Map (KSM)</h1>
		<p class="mb-6 text-center text-gray-700">
			Exposing Kathmandu's pollution problem. Drag the map, see incidents, report what you see!
		</p>
	</div>

	<!-- Map Display -->
	<div class="h-[600px] border border-gray-400 shadow-lg">
		{#if supabase}
			<MapDisplay {supabase} bind:this={mapDisplayComponent} /> <!-- Pass reactive client down -->
		{:else}
			<div class="flex h-[600px] w-full items-center justify-center bg-gray-100">
				<p class="text-gray-500">Initializing Map Data Connection...</p>
			</div>
		{/if}
	</div>
	<!-- Submission Status Message -->
	{#if submissionMessage}
		<div
			class="rounded-md p-4 text-sm"
			class:bg-green-100={!isProcessing && submissionMessage.includes('success')}
			class:text-green-800={!isProcessing && submissionMessage.includes('success')}
			class:bg-blue-100={isProcessing}
			class:text-blue-800={isProcessing}
			class:bg-red-100={!isProcessing &&
				(submissionMessage.includes('Error') ||
					submissionMessage.includes('Failed') ||
					submissionMessage.includes('error'))}
			class:text-red-800={!isProcessing &&
				(submissionMessage.includes('Error') ||
					submissionMessage.includes('Failed') ||
					submissionMessage.includes('error'))}
		>
			{submissionMessage}
		</div>
	{/if}

	<!-- Reporting Form -->
	<div>
		<ReportForm onSubmit={handleReportSubmit} />
	</div>
</section>
