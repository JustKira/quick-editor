<script lang="ts">
	import Button from '$lib/components/ui/button/button.svelte';
	import Input from '$lib/components/ui/input/input.svelte';
	import { onMount } from 'svelte';
	import { Circle, Image, Layer, Line, Stage } from 'svelte-konva';

	let files = $state() as FileList;
	let selectedFile = $derived(files?.[0]);
	let thumbnailUrl = $state<string | null>(null);
	let errorMessage = $state<string | null>(null);

	let image: HTMLImageElement | undefined = $state();
	let imageSize = $state() as { width: number; height: number };

	let drawingActive = $state(false);
	let pointsToDraw = $state([]) as { x: number; y: number }[];

	let polygonName = $state<string | null>(null);
	let polygonTag = $state<string | null>(null);

	let polygons = $state([]) as { name: string; tag: string; points: { x: number; y: number }[] }[];

	// Extract the first frame from a video file
	function extractFirstFrame(videoFile: File): Promise<string> {
		return new Promise((resolve, reject) => {
			// Create video element
			const video = document.createElement('video');
			video.preload = 'metadata';

			// Create object URL for the video file
			const videoUrl = URL.createObjectURL(videoFile);

			// Set up event handlers
			video.onloadeddata = () => {
				// Create canvas to draw the frame
				const canvas = document.createElement('canvas');
				canvas.width = video.videoWidth;
				canvas.height = video.videoHeight;

				// Draw the first frame
				const ctx = canvas.getContext('2d');
				if (!ctx) {
					URL.revokeObjectURL(videoUrl);
					reject(new Error('Failed to get canvas context'));
					return;
				}

				ctx.drawImage(video, 0, 0, canvas.width, canvas.height);

				// Convert canvas to image data URL
				const imageUrl = canvas.toDataURL('image/jpeg');
				const img = document.createElement('img');
				img.src = imageUrl;

				img.onload = () => {
					image = img;
					imageSize = { width: img.width, height: img.height };
				};
				// Clean up
				URL.revokeObjectURL(videoUrl);
				resolve(imageUrl);
			};

			// Handle errors
			video.onerror = () => {
				URL.revokeObjectURL(videoUrl);
				reject(new Error('Error loading video'));
			};

			// Set source and attempt to load
			video.src = videoUrl;
			// Force load of first frame
			video.currentTime = 0.1;
		});
	}

	// Process the selected video file
	async function processVideo() {
		if (!selectedFile || !selectedFile.type.startsWith('video/')) {
			errorMessage = 'Please select a valid video file';
			return;
		}

		try {
			errorMessage = null;
			thumbnailUrl = await extractFirstFrame(selectedFile);
		} catch (error) {
			errorMessage = error instanceof Error ? error.message : 'Unknown error occurred';
			thumbnailUrl = null;
		}
	}

	// Watch for file changes and process automatically
	$effect(() => {
		if (selectedFile && selectedFile.type.startsWith('video/')) {
			processVideo();
		}
	});

	// Clean up any object URLs when component is destroyed
	onMount(() => {
		return () => {
			if (thumbnailUrl && thumbnailUrl.startsWith('blob:')) {
				URL.revokeObjectURL(thumbnailUrl);
			}
		};
	});

	function savePolygon() {
		if (polygonName && polygonTag && pointsToDraw.length > 3) {
			polygons = [
				...polygons,
				{
					name: polygonName,
					tag: polygonTag,
					points: pointsToDraw
				}
			];
			pointsToDraw = [];
			polygonName = null;
			polygonTag = null;
			drawingActive = false;
		}
	}
</script>

<main class="flex flex-col items-center gap-4 p-4">
	<div class="w-full max-w-lg">
		<input type="file" bind:files accept="video/*" />

		{#if selectedFile}
			<p class="mt-2 text-sm">Selected: {selectedFile.name}</p>
		{/if}

		{#if errorMessage}
			<p class="mt-2 text-red-500">{errorMessage}</p>
		{/if}
	</div>
	<div class="flex w-full items-center justify-center gap-2">
		{#if image}
			<div class="flex flex-col gap-2">
				<div class="flex gap-2">
					<Button
						variant={drawingActive ? 'default' : 'secondary'}
						onclick={() => (drawingActive = !drawingActive)}
						>Draw {drawingActive ? 'on' : 'off'}</Button
					>
					<Button
						onclick={() => {
							pointsToDraw.pop();
						}}>Undo</Button
					>

					<div class="flex gap-2">
						<Input type="text" placeholder="Polygon" bind:value={polygonName} />
						<Input type="text" placeholder="Tag" bind:value={polygonTag} />
						<Button onclick={() => savePolygon()}>Save</Button>
					</div>
				</div>
				<Stage
					on:pointerclick={(e) => {
						if (!drawingActive) return;
						const evt = e.detail.currentTarget.getRelativePointerPosition();
						if (evt)
							pointsToDraw.push({
								x: evt.x,
								y: evt.y
							});
					}}
					config={{
						width: imageSize.width,
						height: imageSize.height
					}}
					><Layer>
						<Image
							config={{
								image
							}}
						/>

						{#if drawingActive}
							{#each pointsToDraw as { x, y } (x + y)}
								<Circle
									config={{
										x: x,
										y: y,
										radius: 5,
										fill: 'blue',
										opacity: 1,
										draggable: false
									}}
								/>
							{/each}
						{/if}

						{#each polygons as value (value.name)}
							<Line
								config={{
									points: value.points.flatMap((p) => [p.x, p.y]),
									stroke: 'green',
									strokeWidth: 5,
									closed: true
								}}
							/>
						{/each}
					</Layer>
				</Stage>
			</div>

			<!-- <div class="mt-4 rounded border p-2">
			<img src={thumbnailUrl} alt="First frame thumbnail" class="h-auto max-w-full" />
		</div> -->
		{/if}
		<div class="flex flex-col gap-2">
			<Button onclick={() => navigator.clipboard.writeText(JSON.stringify(polygons))}>Copy</Button>
			<div class="max-h-96 w-full max-w-96 overflow-y-auto rounded-lg border bg-secondary px-2">
				<pre>
                {JSON.stringify(polygons, null, 2)}
            </pre>
			</div>
		</div>
	</div>
</main>
