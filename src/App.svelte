<script>
	import { onMount } from "svelte";

	/** @type HTMLVideoElement */
	let video;

	const sourceAudioUrl = "/assets/appear-online.ogg";
	const impulseResponseUrl = "/assets/FalklandPalaceRoyalTennisCourt.m4a";

	const rampUpSpeed = 1.5;
	const defaultSpeed = 0.7;
	const defaultRampDownDuration = 5000;
	const rampDownDelay = 0;
	let rampDownDuration = defaultRampDownDuration;
	const BPM = 165;
	$: currentBPM = BPM * (video?.playbackRate || defaultSpeed);
	$: beatPeriod = 60000 / currentBPM;

	const audioContext = new AudioContext();
	let sourceNode = new AudioBufferSourceNode(audioContext);
	const gainNode = new GainNode(audioContext);
	const convolverNode = new ConvolverNode(audioContext);
	let boingBuffer = null;

	convolverNode.connect(gainNode);
	gainNode.connect(audioContext.destination);

	fetch(sourceAudioUrl)
		.then((res) => res.arrayBuffer())
		.then((buffer) =>
			audioContext.decodeAudioData(buffer, (decodedBuffer) => {
				sourceNode.buffer = decodedBuffer;
				boingBuffer = decodedBuffer;
			})
		);

	fetch(impulseResponseUrl)
		.then((res) => res.arrayBuffer())
		.then((buffer) => {
			audioContext.decodeAudioData(buffer, (decodedBuffer) => {
				convolverNode.buffer = decodedBuffer;
			});
		});

	let boingCounter = 0;

	onMount(() => {
		video.defaultPlaybackRate = defaultSpeed;
		video.playbackRate = defaultSpeed;
		let previousTime = Date.now();

		function videoTick() {
			let currentTime = Date.now();

			if (currentTime - previousTime > beatPeriod) {
				console.log(`Took ${currentTime - previousTime} ms since last boing`);
				previousTime = currentTime;
				boingCounter++;

				sourceNode = new AudioBufferSourceNode(audioContext);
				sourceNode.buffer = boingBuffer;
				sourceNode.playbackRate.value = Math.random() * 0.5 + 0.5;
				gainNode.gain.value = Math.random() * 0.4 + 0.6;
				sourceNode.connect(convolverNode);
				sourceNode.start(0);
			}

			requestAnimationFrame(videoTick);
		}

		requestAnimationFrame(videoTick);
	});

	let previousRequestId = null;
	let delayTimeout = null;

	function boing() {
		clearTimeout(delayTimeout);
		if (previousRequestId) {
			cancelAnimationFrame(previousRequestId);
		}

		video.playbackRate = rampUpSpeed;

		audioContext.resume();

		let startTime = Date.now();

		function easeRampDown() {
			let now = Date.now();
			let progress = (now - startTime) / rampDownDuration;

			if (progress >= 1) {
				video.playbackRate = defaultSpeed;
				rampDownDuration = defaultRampDownDuration;
				return;
			}

			const reversedEase = 1 - easeOutQuint(progress);
			const playbackRate = reversedEase * rampUpSpeed + defaultSpeed;
			video.playbackRate = playbackRate;
			previousRequestId = requestAnimationFrame(easeRampDown);
		}

		delayTimeout = setTimeout(() => {
			previousRequestId = requestAnimationFrame(easeRampDown);
		}, rampDownDelay);
	}

	function easeOutQuint(x) {
		return 1 - Math.pow(1 - x, 5);
	}
</script>

<div class="container" on:click={boing}>
	<div class="counter">{boingCounter}</div>
	<video autoplay playsinline muted loop bind:this={video} class="h-screen">
		<source src="assets/ls-ap3-v5-60fps-25x2.mp4" />
	</video>
</div>

<style>
	video {
		height: 100%;
		margin: 0 auto;
	}

	.container {
		background: black;
		position: absolute;
		display: flex;
		left: 0;
		right: 0;
		bottom: 0;
		top: 0;
		cursor: pointer;
		overflow: hidden;
	}

	.container video {
		transition: 0.2s ease-out;
	}

	.container:active video {
		transform: scale(1.1);
		transition: 0s;
	}

	.counter {
		position: absolute;
		top: 70%;
		left: 50%;
		transform: translate(-50%, -50%);
		color: white;
		margin: auto;
		font-size: 2.7rem;
		font-family: "Segoe UI", Tahoma, Geneva, Verdana, sans-serif;
		z-index: 50;
	}
</style>
