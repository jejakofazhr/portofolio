<script lang="ts">
	import { onMount } from "svelte";

	const FRAME_COUNT = 96;
	const MAX_FRAME_SIDE = 1280;
	const PLAYBACK_RATE = 2;
	const CAPTURE_RUNS = 3;
	const EASE = 0.12;
	const VIDEO_SRC = "/assets/Video%20Scrub.mp4";
	const END_MARGIN = 0.05;

	let heroEl: HTMLElement | null = $state(null);
	let canvasEl: HTMLCanvasElement | null = $state(null);
	let jakartaTime = $state("--:-- WIB");
	let loadedCount = $state(0);
	let isLoaded = $state(false);

	let loadPercent = $derived(Math.round((loadedCount / FRAME_COUNT) * 100));

	onMount(() => {
		// 1. Live Jakarta Time
		const timeFormat = new Intl.DateTimeFormat("en-GB", {
			timeZone: "Asia/Jakarta",
			hour: "2-digit",
			minute: "2-digit",
		});

		const updateJakartaTime = () => {
			jakartaTime = `${timeFormat.format(new Date())} WIB`;
		};
		updateJakartaTime();
		const timeInterval = setInterval(updateJakartaTime, 30000);

		// 2. Video Scrub Canvas
		if (!heroEl || !canvasEl) {
			return () => clearInterval(timeInterval);
		}

		const ctx = canvasEl.getContext("2d");
		if (!ctx) {
			return () => clearInterval(timeInterval);
		}

		const frames: (ImageBitmap | null)[] = new Array(FRAME_COUNT).fill(
			null,
		);
		let target = 0;
		let current = 0;
		let lastIndex = -1;
		let lastBitmap: ImageBitmap | null = null;
		let dirty = true;
		let rafId: number | null = null;
		let destroyed = false;

		function resizeCanvas() {
			if (!heroEl || !canvasEl) return;
			const dpr = Math.min(window.devicePixelRatio || 1, 2);
			const rect = heroEl.getBoundingClientRect();
			canvasEl.width = Math.round(rect.width * dpr);
			canvasEl.height = Math.round(rect.height * dpr);
			dirty = true;
		}

		function nearestFrame(index: number): ImageBitmap | null {
			for (let d = 0; d < FRAME_COUNT; d++) {
				if (frames[index - d]) return frames[index - d];
				if (frames[index + d]) return frames[index + d];
			}
			return null;
		}

		function draw(bitmap: ImageBitmap) {
			if (!canvasEl || !ctx) return;
			const cw = canvasEl.width;
			const ch = canvasEl.height;
			const scale = Math.max(cw / bitmap.width, ch / bitmap.height);
			const w = bitmap.width * scale;
			const h = bitmap.height * scale;
			ctx.clearRect(0, 0, cw, ch);
			ctx.drawImage(bitmap, (cw - w) / 2, (ch - h) / 2, w, h);
		}

		function tick() {
			if (destroyed) return;
			current += (target - current) * EASE;
			const index = Math.round(current);
			const bitmap = nearestFrame(index);

			if (
				bitmap &&
				(dirty || index !== lastIndex || bitmap !== lastBitmap)
			) {
				draw(bitmap);
				lastIndex = index;
				lastBitmap = bitmap;
				dirty = false;
			}
			rafId = requestAnimationFrame(tick);
		}

		function setTargetFromX(clientX: number) {
			if (!heroEl) return;
			const rect = heroEl.getBoundingClientRect();
			const progress = Math.min(
				Math.max((clientX - rect.left) / rect.width, 0),
				1,
			);
			target = progress * (FRAME_COUNT - 1);
		}

		const onMouseMove = (e: MouseEvent) => setTargetFromX(e.clientX);
		const onTouchStart = (e: TouchEvent) => {
			if (e.touches.length > 0) setTargetFromX(e.touches[0].clientX);
		};
		const onTouchMove = (e: TouchEvent) => {
			if (e.touches.length > 0) setTargetFromX(e.touches[0].clientX);
		};

		heroEl.addEventListener("mousemove", onMouseMove);
		heroEl.addEventListener("touchstart", onTouchStart, { passive: true });
		heroEl.addEventListener("touchmove", onTouchMove, { passive: true });

		window.addEventListener("resize", resizeCanvas);
		resizeCanvas();
		rafId = requestAnimationFrame(tick);

		// Video frame extraction
		const video = document.createElement("video");
		video.src = VIDEO_SRC;
		video.muted = true;
		video.playsInline = true;
		video.preload = "auto";
		video.setAttribute("aria-hidden", "true");
		video.style.cssText =
			"position:fixed;left:0;top:0;width:1px;height:1px;opacity:0;pointer-events:none;";
		document.body.appendChild(video);

		const scratch = document.createElement("canvas");
		const scratchCtx = scratch.getContext("2d");

		const once = (el: EventTarget, type: string) =>
			new Promise((resolve) =>
				el.addEventListener(type, resolve, { once: true }),
			);

		function updateLoader() {
			let count = 0;
			for (let i = 0; i < FRAME_COUNT; i++) {
				if (frames[i]) count++;
			}
			loadedCount = count;
		}

		async function captureFrame(slot: number) {
			if (destroyed || !scratchCtx) return;
			scratchCtx.drawImage(video, 0, 0, scratch.width, scratch.height);
			try {
				const bitmap = await createImageBitmap(scratch);
				if (destroyed) {
					bitmap.close();
					return;
				}
				if (frames[slot]) {
					bitmap.close();
					return;
				}
				frames[slot] = bitmap;
				updateLoader();
				dirty = true;
			} catch (e) {
				console.error("Failed to create bitmap:", e);
			}
		}

		function whenVisible(): Promise<void> {
			if (!document.hidden) return Promise.resolve();
			return new Promise((resolve) => {
				const check = () => {
					if (!document.hidden) {
						document.removeEventListener("visibilitychange", check);
						resolve();
					}
				};
				document.addEventListener("visibilitychange", check);
			});
		}

		function playthrough(endTime: number): Promise<string> {
			return new Promise((resolve) => {
				const pending: Promise<void>[] = [];
				const claimed = new Set<number>();
				let done = false;

				const finish = (result = "done") => {
					if (done) return;
					done = true;
					clearTimeout(safety);
					video.removeEventListener("ended", onEnded);
					video.removeEventListener("pause", onPause);
					video.pause();
					Promise.all(pending).then(() => resolve(result));
				};
				const onEnded = () => finish();
				const onPause = () =>
					finish(document.hidden ? "hidden" : "done");

				const onFrame = (_now: number, meta: { mediaTime: number }) => {
					if (done || destroyed) return;
					const slot = Math.min(
						FRAME_COUNT - 1,
						Math.max(
							0,
							Math.round(
								(meta.mediaTime / endTime) * (FRAME_COUNT - 1),
							),
						),
					);
					if (!frames[slot] && !claimed.has(slot)) {
						claimed.add(slot);
						pending.push(captureFrame(slot));
					}
					if (meta.mediaTime >= endTime) finish();
					else if ("requestVideoFrameCallback" in video) {
						(video as any).requestVideoFrameCallback(onFrame);
					}
				};

				const safety = setTimeout(
					() => finish(document.hidden ? "hidden" : "done"),
					(endTime / PLAYBACK_RATE) * 1000 + 4000,
				);

				video.currentTime = 0;
				once(video, "seeked").then(() => {
					if (destroyed) return;
					if (!frames[0]) {
						claimed.add(0);
						pending.push(captureFrame(0));
					}
					video.playbackRate = PLAYBACK_RATE;
					if ("requestVideoFrameCallback" in video) {
						(video as any).requestVideoFrameCallback(onFrame);
						video
							.play()
							.then(() => {
								video.addEventListener("ended", onEnded);
								video.addEventListener("pause", onPause);
							})
							.catch(() =>
								finish(document.hidden ? "hidden" : "failed"),
							);
					} else {
						finish("unsupported");
					}
				});
			});
		}

		async function fillMissing(endTime: number) {
			for (let slot = 0; slot < FRAME_COUNT; slot++) {
				if (destroyed) return;
				if (frames[slot]) continue;
				await whenVisible();
				video.currentTime = (slot / (FRAME_COUNT - 1)) * endTime;
				await once(video, "seeked");
				await captureFrame(slot);
			}
		}

		async function extractFrames() {
			if (video.readyState < 1) await once(video, "loadedmetadata");

			const scale = Math.min(
				1,
				MAX_FRAME_SIDE / Math.max(video.videoWidth, video.videoHeight),
			);
			scratch.width = Math.round(video.videoWidth * scale);
			scratch.height = Math.round(video.videoHeight * scale);

			const endTime = Math.max(0, video.duration - END_MARGIN);

			if ("requestVideoFrameCallback" in HTMLVideoElement.prototype) {
				let runs = 0;
				while (
					runs < CAPTURE_RUNS &&
					frames.includes(null) &&
					!destroyed
				) {
					await whenVisible();
					const result = await playthrough(endTime);
					if (result === "failed" || result === "unsupported") break;
					if (result === "done") runs++;
				}
			}
			await fillMissing(endTime);

			video.pause();
			video.removeAttribute("src");
			video.load();
			video.remove();
		}

		extractFrames()
			.catch((err) => console.error("Hero video scrub error:", err))
			.finally(() => {
				isLoaded = true;
			});

		return () => {
			destroyed = true;
			clearInterval(timeInterval);
			if (rafId) cancelAnimationFrame(rafId);
			window.removeEventListener("resize", resizeCanvas);
			if (heroEl) {
				heroEl.removeEventListener("mousemove", onMouseMove);
				heroEl.removeEventListener("touchstart", onTouchStart);
				heroEl.removeEventListener("touchmove", onTouchMove);
			}
			video.pause();
			video.removeAttribute("src");
			video.remove();
			frames.forEach((b) => b?.close());
		};
	});
</script>

<section class="hero" id="hero" bind:this={heroEl}>
	<canvas
		class="hero__canvas"
		id="heroCanvas"
		bind:this={canvasEl}
		aria-hidden="true"
	></canvas>
	<div class="hero__shade" aria-hidden="true"></div>

	<div class="hero__content container">
		<div class="hero__main">
			<span class="chip glass"
				><span class="dot"></span>Available for projects, 2026</span
			>
			<h1 class="hero__title">Azhar<br />Danica</h1>
			<p class="hero__intro">
				Designer from Indonesia crafting brands, interfaces and playful
				3D worlds.
			</p>
			<div class="hero__actions">
				<a class="btn btn--light" href="#contact"
					>Get in touch <span class="btn__arrow" aria-hidden="true"
						>→</span
					></a
				>
				<a class="btn glass" href="#work">View work</a>
			</div>
		</div>

		<aside class="hero__stats">
			<div class="stat glass">
				<span class="stat__value">04+</span>
				<span class="stat__label">Years designing</span>
			</div>
			<div class="stat glass">
				<span class="stat__value">30+</span>
				<span class="stat__label">Projects shipped</span>
			</div>
			<div class="stat glass">
				<span class="stat__label">Based in</span>
				<span class="stat__value stat__value--sm">Somakaton, ID</span>
				<span class="stat__time" id="jakartaTime">{jakartaTime}</span>
			</div>
		</aside>
	</div>

	<a class="hero__scroll glass" href="#about" aria-label="Scroll to About">
		<span class="hero__scroll-dot"></span>
	</a>

	<div class="loader" class:is-done={isLoaded} id="loader">
		<div class="loader__pill glass">
			<span class="loader__label">LOADING</span>
			<span
				class="loader__bar"
				role="progressbar"
				aria-label="Loading"
				aria-valuemin="0"
				aria-valuemax="100"
				aria-valuenow={loadPercent}
			>
				<span
					class="loader__fill"
					style="transform: scaleX({loadPercent / 100})"
				></span>
			</span>
			<span class="loader__pct">{loadPercent}%</span>
		</div>
	</div>
</section>
