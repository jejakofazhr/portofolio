<script lang="ts">
	import { onMount } from 'svelte';

	const SPACING = 90;
	const LIFETIME = 1150;
	const TRAIL_MAX_PER_MOVE = 6;
	const TRAIL_IMAGES = [1, 2, 3, 4].map((n) => `/assets/trail/c${n}.png`);

	let trailEl: HTMLElement | null = $state(null);

	onMount(() => {
		if (!trailEl) return;

		// Preload images
		TRAIL_IMAGES.forEach((src) => {
			const img = new Image();
			img.src = src;
		});

		let pressed = false;
		let imageIndex = 0;
		let lastX = 0;
		let lastY = 0;

		const random = (min: number, max: number) => min + Math.random() * (max - min);

		function spawn(x: number, y: number, dirX = 0, dirY = 0) {
			if (!trailEl) return;
			const img = document.createElement('img');
			img.src = TRAIL_IMAGES[imageIndex];
			img.alt = '';
			img.decoding = 'async';
			imageIndex = (imageIndex + 1) % TRAIL_IMAGES.length;

			const width = random(110, 180);
			const tilt = random(-20, 20);
			const spin = tilt < 0 ? -18 : 18;

			img.style.width = `${width}px`;
			img.style.left = `${x + dirX * 14}px`;
			img.style.top = `${y + dirY * 14}px`;
			trailEl.appendChild(img);

			const t = (dy: number, rot: number, scale: number) =>
				`translate(-50%, -50%) translateY(${dy}px) rotate(${rot}deg) scale(${scale})`;

			const anim = img.animate(
				[
					{
						offset: 0,
						transform: t(0, tilt, 0),
						opacity: 1,
						easing: 'cubic-bezier(0.34, 1.7, 0.64, 1)'
					},
					{
						offset: 260 / LIFETIME,
						transform: t(0, tilt, 1),
						opacity: 1,
						easing: 'ease-out'
					},
					{
						offset: 700 / LIFETIME,
						transform: t(0, tilt + spin * 0.15, 0.94),
						opacity: 1,
						easing: 'cubic-bezier(0.55, 0, 0.75, 0.2)'
					},
					{
						offset: 1,
						transform: t(40, tilt + spin, 0.1),
						opacity: 0
					}
				],
				{ duration: LIFETIME, fill: 'forwards' }
			);

			anim.onfinish = () => img.remove();
		}

		function endTrail() {
			if (!pressed) return;
			pressed = false;
			document.body.classList.remove('is-trailing');
		}

		const onPointerDown = (e: PointerEvent) => {
			if (e.button !== 0) return;
			pressed = true;
			lastX = e.clientX;
			lastY = e.clientY;
			document.body.classList.add('is-trailing');
			spawn(lastX, lastY);
		};

		const onPointerMove = (e: PointerEvent) => {
			if (!pressed) return;
			const dx = e.clientX - lastX;
			const dy = e.clientY - lastY;
			const dist = Math.hypot(dx, dy);
			const steps = Math.floor(dist / SPACING);
			if (!steps) return;

			const dirX = dx / dist;
			const dirY = dy / dist;
			const count = Math.min(steps, TRAIL_MAX_PER_MOVE);
			const step = steps > TRAIL_MAX_PER_MOVE ? dist / count : SPACING;

			for (let i = 1; i <= count; i++) {
				spawn(lastX + dirX * step * i, lastY + dirY * step * i, dirX, dirY);
			}
			lastX += dirX * step * count;
			lastY += dirY * step * count;
		};

		window.addEventListener('pointerdown', onPointerDown);
		window.addEventListener('pointermove', onPointerMove);
		window.addEventListener('pointerup', endTrail);
		window.addEventListener('pointercancel', endTrail);
		window.addEventListener('blur', endTrail);

		return () => {
			window.removeEventListener('pointerdown', onPointerDown);
			window.removeEventListener('pointermove', onPointerMove);
			window.removeEventListener('pointerup', endTrail);
			window.removeEventListener('pointercancel', endTrail);
			window.removeEventListener('blur', endTrail);
			document.body.classList.remove('is-trailing');
		};
	});
</script>

<div class="trail" id="trail" bind:this={trailEl} aria-hidden="true"></div>
