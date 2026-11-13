<script lang="ts">
	import { onMount } from 'svelte';
	import { csvParse } from 'd3-dsv';
	import { geoNaturalEarth1, geoPath } from 'd3-geo';
	import { interpolateYlGn } from 'd3-scale-chromatic';
	import { scaleSequential } from 'd3-scale';
	import type { Feature, FeatureCollection, Geometry } from 'geojson';

	type HappinessRow = { country: string; score: number };
	type CountryProperties = { ADMIN?: string; NAME?: string; name?: string };
	type CountryFeature = Feature<Geometry, CountryProperties>;

	let mapContainer: HTMLDivElement;
	let width = $state(960);
	let height = $state(520);
	let countries = $state<CountryFeature[]>([]);
	let happiness = $state(new Map<string, HappinessRow>());
	let selectedCountry = $state<HappinessRow | null>(null);
	let loading = $state(true);
	let error = $state('');

	const aliases: Record<string, string> = {
		'Bolivia (Plurinational State of)': 'Bolivia',
		Czechia: 'Czech Republic',
		Eswatini: 'Swaziland',
		'Iran (Islamic Republic of)': 'Iran',
		'Korea, Republic of': 'South Korea',
		'Moldova, Republic of': 'Moldova',
		'Russian Federation': 'Russia',
		'Taiwan, Province of China': 'Taiwan',
		'Tanzania, United Republic of': 'Tanzania',
		'United States of America': 'United States',
		'Venezuela (Bolivarian Republic of)': 'Venezuela'
	};

	const normalize = (name: string) => aliases[name] ?? name;

	let projection = $derived(geoNaturalEarth1().fitSize([width, height], {
		type: 'FeatureCollection',
		features: countries
	} as FeatureCollection<Geometry, CountryProperties>));
	let path = $derived(geoPath(projection));
	let color = $derived(scaleSequential(interpolateYlGn).domain([2.5, 7.8]));

	function scoreFor(feature: CountryFeature) {
		const name = feature.properties?.ADMIN ?? feature.properties?.NAME ?? feature.properties?.name ?? '';
		return happiness.get(normalize(name));
	}

	function countryName(feature: CountryFeature) {
		return feature.properties?.ADMIN ?? feature.properties?.NAME ?? feature.properties?.name ?? 'Unknown';
	}

	function resize() {
		if (!mapContainer) return;
		width = Math.max(320, mapContainer.clientWidth);
		height = Math.max(260, Math.min(560, width * 0.54));
	}

	async function loadData() {
		try {
			const [csvResponse, geographyResponse] = await Promise.all([
				fetch('/2019.csv'),
				fetch('https://raw.githubusercontent.com/datasets/geo-countries/master/data/countries.geojson')
			]);
			if (!csvResponse.ok || !geographyResponse.ok) throw new Error('The map data could not be loaded.');

			const parsed = csvParse(await csvResponse.text(), (row) => ({
				country: row['Country or region'] ?? '',
				score: Number(row.Score)
			}));
			happiness = new Map(
			parsed
				.filter((row) => row.country && Number.isFinite(row.score))
				.map((row) => [row.country, row])
			);

			const geography = (await geographyResponse.json()) as FeatureCollection<Geometry, CountryProperties>;
			countries = geography.features as CountryFeature[];
			resize();
		} catch (loadError) {
			error = loadError instanceof Error ? loadError.message : 'The map data could not be loaded.';
		} finally {
			loading = false;
		}
	}

	onMount(() => {
		resize();
		const observer = new ResizeObserver(resize);
		observer.observe(mapContainer);
		loadData();
		return () => observer.disconnect();
	});
</script>

<section class="map-card" aria-labelledby="map-title">
	<div class="heading">
		<div>
			<p class="eyebrow">World Happiness Report · 2019</p>
			<h1 id="map-title">How happy is the world?</h1>
			<p class="subtitle">Countries are shaded by their average happiness score.</p>
		</div>
		{#if selectedCountry}
			<div class="selected-country">
				<span>{selectedCountry.country}</span>
				<strong>{selectedCountry.score.toFixed(3)}</strong>
			</div>
		{/if}
	</div>

	<div class="map-wrap" bind:this={mapContainer}>
		{#if loading}
			<p class="status">Loading the world map…</p>
		{:else if error}
			<p class="status error">{error}</p>
		{:else}
			<svg viewBox={`0 0 ${width} ${height}`} role="img" aria-label="World map colored by 2019 happiness score">
				<rect class="ocean" width={width} height={height} rx="16" />
				{#each countries as country}
					{@const score = scoreFor(country)}
					<path
						class:missing={!score}
						d={path(country) ?? ''}
						fill={score ? color(score.score) : '#d9e2e7'}
						role="button"
						tabindex={score ? 0 : -1}
						aria-label={score ? `${countryName(country)}: ${score.score.toFixed(3)}` : countryName(country)}
						onmouseenter={() => (selectedCountry = score ?? null)}
						onfocus={() => (selectedCountry = score ?? null)}
						onmouseleave={() => (selectedCountry = null)}
						onblur={() => (selectedCountry = null)}
					/>
				{/each}
			</svg>
		{/if}
	</div>

	<div class="footer">
		<div class="legend" aria-label="Happiness score legend">
			<span>Lower</span>
			<div class="legend-gradient"></div>
			<span>Higher</span>
		</div>
		<p class="source">Source: 2019.csv · Hover over a country to inspect its score.</p>
	</div>
</section>

<style>
	:global(body) { margin: 0; background: #f2f7f6; font-family: Inter, ui-sans-serif, system-ui, sans-serif; color: #173c3a; }
	.map-card { width: min(1180px, calc(100% - 32px)); margin: 32px auto; padding: clamp(22px, 4vw, 48px); box-sizing: border-box; background: #fff; border: 1px solid #dbe9e5; border-radius: 24px; box-shadow: 0 18px 50px rgba(23, 60, 58, 0.08); }
	.heading, .footer { display: flex; align-items: center; justify-content: space-between; gap: 24px; }
	.eyebrow { margin: 0 0 10px; color: #de7b49; font-size: .75rem; font-weight: 800; letter-spacing: .12em; text-transform: uppercase; }
	h1 { margin: 0; font-size: clamp(2rem, 5vw, 4.4rem); line-height: .95; letter-spacing: -.06em; }
	.subtitle { margin: 14px 0 0; color: #66817d; font-size: 1rem; }
	.selected-country { min-width: 130px; padding: 14px 16px; border-radius: 14px; background: #eef7ef; text-align: right; }
	.selected-country span, .selected-country strong { display: block; }
	.selected-country span { color: #66817d; font-size: .8rem; }
	.selected-country strong { margin-top: 3px; font-size: 1.55rem; }
	.map-wrap { position: relative; min-height: 260px; margin-top: 32px; }
	svg { display: block; width: 100%; height: auto; }
	.ocean { fill: #eaf4f4; }
	path { stroke: #fff; stroke-width: .65px; vector-effect: non-scaling-stroke; transition: opacity 120ms ease, stroke 120ms ease; }
	path:hover, path:focus { stroke: #173c3a; stroke-width: 1.5px; outline: none; }
	path.missing { cursor: default; }
	.status { position: absolute; inset: 50% auto auto 50%; transform: translate(-50%, -50%); color: #66817d; }
	.status.error { color: #b34e3c; }
	.legend { display: flex; align-items: center; gap: 8px; color: #66817d; font-size: .78rem; }
	.legend-gradient { width: 130px; height: 10px; border-radius: 99px; background: linear-gradient(90deg, #ffffe5, #d9f0a3, #78c679, #238443, #004529); }
	.source { margin: 0; color: #8ca19d; font-size: .78rem; text-align: right; }
	@media (max-width: 640px) {
		.map-card { width: calc(100% - 16px); margin: 8px auto; border-radius: 18px; }
		.heading, .footer { align-items: flex-start; flex-direction: column; }
		.selected-country { width: 100%; box-sizing: border-box; text-align: left; }
		.source { text-align: left; }
	}
</style>
