<script lang="ts">
	import { onMount } from "svelte";

	let marqueeEl: HTMLElement | null = $state(null);
	let row1El: HTMLElement | null = $state(null);
	let row2El: HTMLElement | null = $state(null);

	onMount(() => {
		const reduceMotion = window.matchMedia(
			"(prefers-reduced-motion: reduce)",
		).matches;
		if (reduceMotion || !marqueeEl || !row1El || !row2El) return;

		const MARQUEE_SPEED = 60;
		const rowEls = [row1El, row2El];
		const rows = rowEls.map((row) => {
			const track = row.querySelector<HTMLElement>(".marquee__track")!;
			const group = track.querySelector<HTMLElement>(".marquee__group")!;
			return {
				track,
				group,
				dir: row.classList.contains("marquee--reverse") ? 1 : -1,
				width: 0,
				offset: 0,
			};
		});

		function measureMarquee() {
			for (const r of rows) {
				r.track
					.querySelectorAll(".marquee__group[data-clone]")
					.forEach((c) => c.remove());
				r.width = r.group.getBoundingClientRect().width;
				if (!r.width || !r.track.parentElement) continue;
				const needed =
					Math.ceil(r.track.parentElement.offsetWidth / r.width) + 1;
				for (let i = 0; i < needed; i++) {
					const clone = r.group.cloneNode(true) as HTMLElement;
					clone.dataset.clone = "";
					r.track.appendChild(clone);
				}
				r.offset %= r.width;
			}
		}

		let visible = true;
		let boost = 0;
		let lastScrollY = window.scrollY;
		let lastTime = performance.now();
		let rafId: number | null = null;

		const observer = new IntersectionObserver(([entry]) => {
			visible = entry.isIntersecting;
		});
		observer.observe(marqueeEl);

		function marqueeTick(now: number) {
			const dt = Math.min((now - lastTime) / 1000, 0.1);
			lastTime = now;

			const scrollY = window.scrollY;
			const velocity = dt > 0 ? Math.abs(scrollY - lastScrollY) / dt : 0;
			lastScrollY = scrollY;
			const boostTarget = Math.min(velocity * 0.4, 900);
			boost += (boostTarget - boost) * 0.08;

			if (visible) {
				const step = (MARQUEE_SPEED + boost) * dt;
				for (const r of rows) {
					if (!r.width) continue;
					r.offset = (r.offset + step) % r.width;
					const x = r.dir < 0 ? -r.offset : r.offset - r.width;
					r.track.style.transform = `translate3d(${x}px, 0, 0)`;
				}
			}
			rafId = requestAnimationFrame(marqueeTick);
		}

		measureMarquee();
		document.fonts.ready.then(measureMarquee);
		window.addEventListener("resize", measureMarquee);
		rafId = requestAnimationFrame(marqueeTick);

		return () => {
			if (rafId) cancelAnimationFrame(rafId);
			observer.disconnect();
			window.removeEventListener("resize", measureMarquee);
		};
	});
</script>

<section class="about" id="about">
	<div
		class="marquee"
		bind:this={marqueeEl}
		aria-label="Brand Identity, UI / UX, 3D Design, Motion. Azhar Danica, Graphic Designer, Indonesia, since 2018."
	>
		<div class="marquee__row" bind:this={row1El} aria-hidden="true">
			<div class="marquee__track">
				<div class="marquee__group">
					<span>Brand Identity</span><span class="marquee__sep"
						>✦</span
					>
					<span class="outline">UI / UX</span><span
						class="marquee__sep">✦</span
					>
					<span>3D Design</span><span class="marquee__sep">✦</span>
					<span class="outline">Motion</span><span
						class="marquee__sep">✦</span
					>
				</div>
			</div>
		</div>
		<div
			class="marquee__row marquee--reverse"
			bind:this={row2El}
			aria-hidden="true"
		>
			<div class="marquee__track">
				<div class="marquee__group">
					<span>Azhar Danica</span><span class="marquee__sep">✦</span>
					<span class="outline">Graphic Designer</span><span
						class="marquee__sep">✦</span
					>
					<span>Indonesia</span><span class="marquee__sep">✦</span>
					<span class="outline">Since 2018</span><span
						class="marquee__sep">✦</span
					>
				</div>
			</div>
		</div>
	</div>

	<div class="about__body container">
		<p class="eyebrow">(01) About</p>
		<p class="about__text">
			I'm Azhar Danica, a multidisciplinary designer working out of
			Somakaton. I build
			<strong>identities people remember</strong>, interfaces that feel
			effortless and 3D scenes with a sense of play, for studios and
			startups across Asia and beyond.
		</p>
		<ul class="about__tags">
			<li class="tag">Branding</li>
			<li class="tag">Product Design</li>
			<li class="tag">Blender / C4D</li>
			<li class="tag">Figma</li>
			<li class="tag">Art Direction</li>
		</ul>
	</div>
</section>
