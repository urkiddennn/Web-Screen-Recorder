<script lang="ts">
	type State = 'ready' | 'ready.countdown' | 'recording';
	type Optional<T> = T | undefined;

	export let width: Optional<number> = undefined;
	export let height: Optional<number> = undefined;
	export let audio = true;
	export let frameRate = 60;

	let state: State = 'ready';
	let recorder: MediaRecorder;
	let stream: MediaStream;
	let videoChunks = [] as Blob[];

	let timerInterval: any; // or number
	let timer = 3;

	async function startTimer() {
		state = 'ready.countdown';

		return new Promise((resolve) => {
			timerInterval = setInterval(() => {
				timer--;
				if (timer == 0) {
					clearInterval(timerInterval);
					resolve(timer);
				}
			}, 1000);
		});
	}

	function downloadRecording(videoChunks: Blob[]) {
		const blob = new Blob(videoChunks, { type: 'video/webm; codecs=vp9' });
		const url = URL.createObjectURL(blob);
		const a = document.createElement('a');
		a.href = url;
		a.download = 'video.webm';
		document.body.appendChild(a);
		a.click();
		document.body.removeChild(a);
		URL.revokeObjectURL(url); // Clean up
	}

	function stopRecording() {
		state = 'ready';
		stream.getTracks().forEach((track) => track.stop());
		if (recorder) recorder.stop(); // Ensure recorder stops
		clearInterval(timerInterval);
		timer = 3;
	}

	function startRecording() {
		state = 'recording';
		recorder = new MediaRecorder(stream, { mimeType: 'video/webm; codecs=vp9' });

		recorder.start(); // Properly invoke recorder.start()

		recorder.addEventListener('dataavailable', (event) => {
			videoChunks.push(event.data); // Collect video chunks
		});

		recorder.addEventListener('stop', () => {
			downloadRecording(videoChunks); // Trigger download when recording stops
			videoChunks = []; // Reset video chunks
		});
	}

	async function prepareRecording() {
		try {
			stream = await navigator.mediaDevices.getDisplayMedia({
				video: { width, height, frameRate },
				audio,
				// @ts-ignore
				preferCurrentTab: false
			});

			stream.addEventListener('inactive', stopRecording);

			await startTimer();
			startRecording();
		} catch (error) {
			console.log(`Error: ${error}`);
		}
	}

	function handleKeyDown(event: KeyboardEvent) {
		switch (event.key.toLowerCase()) {
			case 'r':
				if (state === 'ready') {
					console.log('prepare recording');
					prepareRecording();
				}
				break;
			case 'e':
				if (state !== 'ready') {
					console.log('stopping');
					stopRecording();
				}
				break;
		}
	}
</script>

<svelte:window on:keydown={handleKeyDown} />

{#if state.includes('ready')}
	<div class="main">
		<h1 class="name">✨ Recorden</h1>
		<div class="recorder" data-state={state}>
			<button on:click={prepareRecording} class="record">
				<div class="circle">
					{#if state == 'ready.countdown'}
						{timer}
					{/if}
				</div>
			</button>
		</div>
	</div>
{/if}
<div class="info">
	{#if state === 'ready'}
		<p class="shortcut">Shift + r</p>
		<p class="description">To Start recording</p>
	{/if}
	{#if state === 'ready.countdown'}
		<p class="shortcut">Shift + s</p>
		<p class="description">To Stop recording</p>
	{/if}
</div>

<style>
	.recorder {
		--txt-clr: hsl(0, 0%, 98%);
		--circle-bg-clr: hsl(0, 100%, 60%);
		--circle-border-clr: hsl(0 0% 98%);

		position: absolute;
		left: 50%;
		bottom: 40px;
		color: var(--txt-clr);
		text-align: center;
		z-index: 10;
	}
	[data-state='ready.countdowm'] {
		& .circle {
			--txt-clr: hsl(0 0% 10%);
			--circle-bg-clr: hsl(0 0% 98%);
		}
	}
	.main {
		height: 100%;
		height: 100vh;
		background-color: hsl(0 0% 10%);
		overflow: hidden;
	}
	.record {
		width: 80px;
		height: 80px;
		padding: 4px;
		border: 4px solid var(--circle-border-clr);
		border-radius: 50%;
	}
	.name {
		color: hsl(0, 0%, 98%);
	}
	.circle {
		width: 100%;
		height: 100%;
		display: grid;
		place-content: center;
		font-size: 2rem;
		font-weight: 700;
		color: var(--txt-clr);
		background: var(--circle-bg-clr);
		border-radius: 50%;
		transition: background-color 0.6s;

		&:hover {
			--circle-bg-clr: hsl(0 100% 64%);
		}
	}
	.info {
		margin-block-start: 1rem;
		& .shortcut {
			font-weight: 700;
		}
		& .description {
			margin-block-start: 4px;
			opacity: 0.6;
		}
	}
</style>
