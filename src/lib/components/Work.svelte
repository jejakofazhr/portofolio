<script lang="ts">
	import { onMount } from 'svelte';

	let workEl: HTMLElement | null = $state(null);
	let workTrackEl: HTMLElement | null = $state(null);
	let workProgressEl: HTMLElement | null = $state(null);
	let currentNum = $state('01');

	const projects = [
		{
			num: '01',
			title: 'Kopi Senja',
			cat: 'Brand identity · Packaging',
			year: '2025',
			classModifier: 'project--kopi'
		},
		{
			num: '02',
			title: 'Lumen Bank',
			cat: 'Mobile app · UI / UX',
			year: '2025',
			classModifier: 'project--lumen'
		},
		{
			num: '03',
			title: 'Pixel Pals',
			cat: '3D characters · Motion',
			year: '2024',
			classModifier: 'project--pixel'
		},
		{
			num: '04',
			title: 'Batik Now',
			cat: 'Editorial · Art direction',
			year: '2024',
			classModifier: 'project--batik'
		},
		{
			num: '05',
			title: 'Orbit OS',
			cat: 'Product design · Web',
			year: '2023',
			classModifier: 'project--orbit'
		}
	];

	onMount(() => {
		if (!workEl || !workTrackEl || !workProgressEl) return;

		let distance = 0;
		const cards = Array.from(workTrackEl.querySelectorAll<HTMLElement>('.project'));

		function measureWork() {
			if (!workEl || !workTrackEl) return;
			distance = Math.max(0, workTrackEl.scrollWidth - window.innerWidth);
			workEl.style.height = `${distance + window.innerHeight}px`;
			updateWork();
		}

		function updateWork() {
			if (!workEl || !workTrackEl || !workProgressEl) return;
			const top = workEl.getBoundingClientRect().top;
			const progress = distance ? Math.min(Math.max(-top / distance, 0), 1) : 0;

			workTrackEl.style.transform = `translate3d(${-progress * distance}px, 0, 0)`;
			workProgressEl.style.width = `${progress * 100}%`;

			const center = window.innerWidth * progress + progress * distance;
			let current = 0;
			let best = Infinity;
			cards.forEach((card, i) => {
				const d = Math.abs(card.offsetLeft + card.offsetWidth / 2 - center);
				if (d < best) {
					best = d;
					current = i;
				}
			});
			currentNum = String(current + 1).padStart(2, '0');
		}

		window.addEventListener('scroll', updateWork, { passive: true });
		window.addEventListener('resize', measureWork);
		document.fonts.ready.then(measureWork);
		measureWork();

		return () => {
			window.removeEventListener('scroll', updateWork);
			window.removeEventListener('resize', measureWork);
		};
	});
</script>

<section class="work" id="work" data-nav="light" bind:this={workEl}>
	<div class="work__sticky">
		<div class="work__head container">
			<p class="eyebrow">(02) Selected Work</p>
			<p class="work__counter"><span id="workCurrent">{currentNum}</span> / 05</p>
		</div>

		<div class="work__track" id="workTrack" bind:this={workTrackEl}>
			<div class="work__intro">
				<h2 class="work__title">Selected<br />Projects</h2>
				<p>Scroll to explore a few favourites from the last couple of years.</p>
			</div>

			{#each projects as project (project.num)}
				<article class="project {project.classModifier}">
					<div class="project__art"></div>
					<span class="project__num">{project.num}</span>
					<div class="project__info glass">
						<div>
							<h3 class="project__title">{project.title}</h3>
							<p class="project__cat">{project.cat}</p>
						</div>
						<span class="project__year">{project.year}</span>
					</div>
				</article>
			{/each}
		</div>

		<div class="work__progress" aria-hidden="true">
			<span id="workProgress" bind:this={workProgressEl}></span>
		</div>
	</div>
</section>
