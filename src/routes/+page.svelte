<script lang="ts">
	import Nav from "#lib/components/Nav.svelte";
	import Hero from "#lib/components/Hero.svelte";
	import About from "#lib/components/About.svelte";
	import Work from "#lib/components/Work.svelte";
	import Testimonials from "#lib/components/Testimonials.svelte";
	import Footer from "#lib/components/Footer.svelte";
	import Trail from "#lib/components/Trail.svelte";
</script>

<svelte:head>
	<title>Azhar Danica Budi Zulfalillah — Graphic Design</title>
	<meta
		name="description"
		content="Azhar Danica Budi Zulfalillah is a graphic designer from Somakaton, Photographer, Videographer and Web Developer."
	/>
</svelte:head>

<Nav />

<main>
	<Hero />
	<About />
	<Work />
	<Testimonials />
</main>

<Footer />

<Trail />
