<script lang="ts">
	import { onMount } from "svelte";

	let navEl: HTMLElement | null = $state(null);
	let isDark = $state(false);

	onMount(() => {
		let navQueued = false;

		function updateNav() {
			navQueued = false;
			if (!navEl) return;
			const navRect = navEl.getBoundingClientRect();
			const y = navRect.top + navRect.height / 2;

			const lightSections = Array.from(
				document.querySelectorAll<HTMLElement>('[data-nav="light"]'),
			);
			const overLight = lightSections.some((section) => {
				const rect = section.getBoundingClientRect();
				return rect.top <= y && rect.bottom >= y;
			});

			isDark = overLight;
		}

		function queueNav() {
			if (navQueued) return;
			navQueued = true;
			requestAnimationFrame(updateNav);
		}

		window.addEventListener("scroll", queueNav, { passive: true });
		window.addEventListener("resize", queueNav);
		updateNav();

		return () => {
			window.removeEventListener("scroll", queueNav);
			window.removeEventListener("resize", queueNav);
		};
	});
</script>

<nav
	bind:this={navEl}
	class="nav glass"
	class:nav--dark={isDark}
	aria-label="Main"
>
	<a class="nav__logo" href="#hero"><b>Azhar</b> <span>Dev</span></a>
	<ul class="nav__links">
		<li><a href="#about">About</a></li>
		<li><a href="#work">Work</a></li>
		<li><a href="#testimonials">Words</a></li>
	</ul>
	<a class="btn btn--light btn--sm" href="#contact">Contact</a>
</nav>
