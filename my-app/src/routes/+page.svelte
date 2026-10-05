<script lang="ts">
	import { onMount, untrack } from 'svelte';
	import { fade } from 'svelte/transition';
	import type { TransitionConfig } from 'svelte/transition';
	import TypewriterMessage from '#lib/TypewriterMessage.svelte';

	const scenes = [
		{ name: 'Day', time: '3 PM', hour: '15', gradient: 'linear-gradient(110deg, #5de0e6, #004aad)', layers: ['day_sun', 'day_clouds'] },
		{ name: 'Evening', time: '6 PM', hour: '18', gradient: 'linear-gradient(110deg, #ff914d, #8c52ff)', layers: ['sunset_sun', 'sunset_clouds2', 'sunset_clouds1'] },
		{ name: 'Night', time: '9 PM', hour: '21', gradient: 'linear-gradient(110deg, #3533cd, #000000)', layers: ['night_stars', 'night_clouds', 'night_moon'] }
	];
	const messages = [
		'Today is shaping up to be a beautiful, sunny day with clear skies and plenty of sunshine. Consider taking a walk in the park, going for a bike ride, or enjoying lunch outdoors. Don’t forget your sunglasses and sunscreen if you’ll be out for a while. Enjoy the sunshine!',
		'The evening is the perfect time to slow down and unwind after a busy day. Try visiting a cozy café or grab dinner with friends. If you’d rather stay in, put on some music, read a book, or enjoy a favorite show. Take the evening at your own pace and give yourself some time to recharge.',
		'A cool night is a great chance to get out and explore somewhere new, whether it’s a lively neighborhood, a late-night café, or a scenic spot around town. Bring a jacket or an extra layer to stay comfortable as temperatures drop. If you’re heading somewhere unfamiliar, stick to well-lit areas and stay aware of your surroundings. Enjoy the cooler air and make the most of the night!'
	];
	let active = $state(0);
	let temperatures = $state<(number | null)[]>([null, null, null]);
	let currentTemperature = $state<number | null>(null);
	let displayedTemperature = $state<number | null>(null);
	let location = $state('Your location');
	let date = $state('');
	let status = $state('Finding your location…');
	let loading = $state(true);
	let transitioning = $state(false);
	let reducedMotion = $state(false);
	let frame = 0;
	let transitionTimer: ReturnType<typeof setTimeout>;
	let controller: AbortController | undefined;
	let disposed = false;
	const scene = $derived(scenes[active]);

	const easeInOutSine = (t: number) => (1 - Math.cos(Math.PI * t)) / 2;
	const layerDuration = 2000;
	const layerEntryDelay = 700;
	const layerStagger = 100;

	// Keep the number tied to the selected forecast, including the night-to-day wrap.
	$effect(() => {
		const target = temperatures[active];
		reducedMotion;
		untrack(() => animateTemperature(target));
	});

	function layerTransition(node: Element, { index, entering }: { index: number; entering: boolean }): TransitionConfig {
		if (node.matches('.day_sun, .sunset_sun, .night_moon')) {
			return {
				duration: reducedMotion ? 0 : layerDuration,
				delay: reducedMotion || !entering ? 0 : 500,
				easing: easeInOutSine,
				css: (t) => `opacity: ${t};`
			};
		}
		// Use viewport distance so every layer travels completely beyond the left edge.
		const distance = window.innerWidth + node.getBoundingClientRect().width;
		return {
			duration: reducedMotion ? 0 : layerDuration,
			delay: reducedMotion ? 0 : (entering ? layerEntryDelay : 0) + index * layerStagger,
			easing: easeInOutSine,
			css: (t) => `transform: translate3d(${-distance * (1 - t)}px, 0, 0);`
		};
	}

	function animateTemperature(target: number | null) {
		cancelAnimationFrame(frame);
		if (target === null || displayedTemperature === null || reducedMotion) {
			displayedTemperature = target;
			return;
		}
		const start = displayedTemperature;
		const duration = Math.max(1900, Math.abs(target - start) * 20);
		const started = performance.now();
		function tick(now: number) {
			const progress = Math.min((now - started) / duration, 1);
			displayedTemperature = Math.round(start + (target! - start) * progress);
			if (progress < 1) frame = requestAnimationFrame(tick);
		}
		frame = requestAnimationFrame(tick);
	}

	function next() {
		if (transitioning) return;
		active = (active + 1) % scenes.length;
		transitioning = true;
		transitionTimer = setTimeout(() => { transitioning = false; }, reducedMotion ? 0 : layerDuration + layerEntryDelay + 3 * layerStagger);
	}

	function fail(message = '') {
		if (disposed) return;
		loading = false;
		status = message;
	}

	function loadWeather() {
		loading = true;
		status = 'Finding your location…';
		if (!navigator.geolocation) {
			fail('Your browser does not support location access.');
			return;
		}
		navigator.geolocation.getCurrentPosition(async (position) => {
			if (disposed) return;
			const { latitude, longitude } = position.coords;
			status = 'Fetching today’s weather…';
			controller?.abort();
			controller = new AbortController();
			const timeout = setTimeout(() => controller?.abort(), 15000);
			try {
				const params = new URLSearchParams({
					latitude: String(latitude), longitude: String(longitude),
					current: 'temperature_2m', hourly: 'temperature_2m',
					temperature_unit: 'fahrenheit', timezone: 'auto', forecast_days: '1'
				});
				const response = await fetch(`https://api.open-meteo.com/v1/forecast?${params}`, { signal: controller.signal });
				if (!response.ok) throw new Error('Weather request failed');
				const data = await response.json();
				if (disposed) return;
				if (!Number.isFinite(data.current?.temperature_2m) || typeof data.current?.time !== 'string' || !Array.isArray(data.hourly?.time) || !Array.isArray(data.hourly?.temperature_2m)) {
					throw new Error('Incomplete weather data');
				}
				const today = data.current.time.slice(0, 10);
				const samples = scenes.map(({ hour }) => {
					const index = data.hourly.time.indexOf(`${today}T${hour}:00`);
					const value = data.hourly.temperature_2m[index];
					if (!Number.isFinite(value)) throw new Error('Missing hourly forecast');
					return Math.round(value);
				});
				temperatures = samples;
				currentTemperature = Math.round(data.current.temperature_2m);
				location = `${Math.abs(latitude).toFixed(2)}° ${latitude >= 0 ? 'N' : 'S'}, ${Math.abs(longitude).toFixed(2)}° ${longitude >= 0 ? 'E' : 'W'}`;
				date = new Intl.DateTimeFormat('en-US', { month: 'numeric', day: 'numeric', year: 'numeric', timeZone: 'UTC' }).format(new Date(`${today}T12:00:00Z`));
				loading = false;
				status = '';
			} catch {
				fail();
			} finally {
				clearTimeout(timeout);
			}
		}, (reason) => {
			fail(reason.code === 1 ? 'Allow location access to see your local weather.' : 'Could not find your location.');
		}, { enableHighAccuracy: false, timeout: 10000, maximumAge: 300000 });
	}

	onMount(() => {
		const preference = window.matchMedia('(prefers-reduced-motion: reduce)');
		reducedMotion = preference.matches;
		const updatePreference = () => { reducedMotion = preference.matches; };
		preference.addEventListener('change', updatePreference);
		loadWeather();
		return () => {
			disposed = true;
			controller?.abort();
			cancelAnimationFrame(frame);
			clearTimeout(transitionTimer);
			preference.removeEventListener('change', updatePreference);
		};
	});
</script>

<svelte:head>
	<title>Weather · Day, Evening & Night</title>
	<meta name="description" content="Your local weather, from afternoon sunshine to evening skies and a quiet night." />
	{#each scenes.flatMap(({ layers }) => layers) as asset}
		<link rel="preload" as="image" href={`/images/assets/${asset}.png`} />
	{/each}
</svelte:head>

<main>
	<section class="weather" aria-label={`${scene.name} weather forecast`} aria-busy={loading}>
		<div class="sky" aria-hidden="true">
			{#key active}
				<div class="gradient" style:background={scene.gradient} in:fade|global={{ duration: reducedMotion ? 0 : layerDuration, easing: easeInOutSine }} out:fade|global={{ duration: reducedMotion ? 0 : layerDuration, easing: easeInOutSine }}></div>
			{/key}
		</div>
		<div class="artwork" aria-hidden="true">
			{#key active}
				{#each scene.layers as layer, index}
					<div class="layer {layer}" in:layerTransition|global={{ index, entering: true }} out:layerTransition|global={{ index, entering: false }}>
						<img src={`/images/assets/${layer}.png`} alt="" draggable="false" />
					</div>
				{/each}
			{/key}
		</div>
		<div class="details">
			<header>
				<p class="period" aria-live="polite">{scene.name} <span>· {scene.time}</span></p>
				<h1 aria-label={displayedTemperature === null ? 'Temperature unavailable' : `${temperatures[active]} degrees Fahrenheit`}><span>{displayedTemperature ?? '—'}</span><span class="unit">° F</span></h1>
				<p class="location">{location}</p>
				<p class="date">{date || 'Today'}</p>
			</header>
			<div class="glass message-box">
				<TypewriterMessage text={messages[active]} {reducedMotion} />
			</div>
			<button class="glass next" onclick={next} disabled={transitioning} aria-label={`Next: ${scenes[(active + 1) % scenes.length].name}`}>Next</button>
		</div>
	</section>
	<div class="weather-note" aria-live="polite">
		{#if status}
			<p>{status}</p>
		{:else if currentTemperature !== null}
			<p>Currently {currentTemperature}° F <span>·</span> Forecast times are local to your location.</p>
		{/if}
	</div>
	<a class="credit" href="https://open-meteo.com/" target="_blank" rel="noreferrer">Weather data by Open-Meteo</a>
</main>

<style>
	:global(*) { box-sizing: border-box; }
	:global(body) { margin: 0; background: #004aad; font-family: Arial, Helvetica, sans-serif; -webkit-font-smoothing: antialiased; }
	:global(button) { font: inherit; }
	main { height: 100dvh; width: 100%; position: relative; overflow: hidden; color: white; }
	.weather { width: 100%; height: 100%; position: relative; isolation: isolate; color: white; }
	.sky { position: absolute; inset: 0; overflow: hidden; z-index: -2; background: #004aad; }
	.gradient { position: absolute; inset: 0; }
	.artwork { position: absolute; inset: -32% -12% -25% -38%; pointer-events: none; z-index: -1; overflow: hidden; }
	.layer { position: absolute; width: 65%; aspect-ratio: 1; }
	.layer img { display: block; width: 100%; height: 100%; object-fit: contain; user-select: none; }
	.day_sun { width: 35%; left: 18%; top: 5%; }
	.day_clouds { width: 32%; left: 23%; top: 44%; }
	.day_clouds img { opacity: 0.8; }
	.sunset_sun { width: 35%; left: 18%; top: 8%; }
	.sunset_clouds2 { width: 53%; left: 3%; top: 39%; }
	.sunset_clouds1 { width: 33%; left: 36%; top: 27%; }
	.night_stars { width: 36%; left: 31%; top: 35%; }
	.night_clouds { width: 35%; left: 13%; top: 30%; }
	.night_moon { width: 21%; left: 18.5%; top: 18%; }
	.night_moon img { filter: drop-shadow(0 0 22px rgb(235 239 255 / 65%)) drop-shadow(0 0 65px rgb(167 181 255 / 40%)); }
	.details { position: absolute; top: max(7%, env(safe-area-inset-top)); right: 5%; bottom: 15%; width: 36%; display: flex; flex-direction: column; gap: min(2dvh, 22px); container-type: inline-size; }
	header { text-align: right; min-width: 0; }
	.period { margin: 0 0 1dvh; font-size: min(4.5cqi, 2.5dvh, 22px); font-weight: 500; letter-spacing: 0.04em; }
	.period span { opacity: 0.8; }
	h1 { margin: 0; font-size: min(26cqi, 19dvh, 190px); line-height: 1; font-weight: 700; letter-spacing: -0.045em; white-space: nowrap; font-variant-numeric: tabular-nums; }
	.unit { letter-spacing: -0.06em; }
	.location { margin: 1dvh 0 0.5dvh; font-size: min(5cqi, 3dvh, 28px); font-weight: 600; overflow-wrap: anywhere; }
	.date { margin: 0; font-size: min(3.5cqi, 2dvh, 16px); }
	.glass { position: relative; background: linear-gradient(135deg, rgb(255 255 255 / 44%), rgb(255 255 255 / 15%) 60%, rgb(255 255 255 / 28%)); border: 1px solid rgb(255 255 255 / 55%); backdrop-filter: blur(38px) saturate(180%); -webkit-backdrop-filter: blur(38px) saturate(180%); }
	.message-box { flex: 1; min-height: 0; border-radius: min(6cqi, 36px); overflow: hidden; }
	.next { flex-shrink: 0; width: 100%; height: 10%; min-height: min(44px, 9dvh); border-radius: 999px; color: white; font-size: min(5cqi, 3dvh, 24px); cursor: pointer; transition: background 250ms ease, color 250ms ease; }
	.next:hover { background: rgb(255 255 255 / 92%); color: #111; }
	.next:focus-visible, .credit:focus-visible { outline: 2px solid currentColor; outline-offset: 5px; }
	.next:disabled { cursor: progress; }
	.weather-note { position: absolute; bottom: max(6%, calc(env(safe-area-inset-bottom) + 24px)); left: 5%; right: 5%; text-align: center; font-size: min(3vw, 2dvh, 15px); color: rgb(255 255 255 / 90%); line-height: 1.5; text-shadow: 0 1px 8px rgb(0 0 0 / 30%); }
	.weather-note p { margin: 0; }
	.weather-note span { margin: 0 5px; }
	.credit { position: absolute; bottom: max(2%, env(safe-area-inset-bottom)); left: 5%; right: 5%; text-align: center; font-size: min(2.7vw, 1.7dvh, 12px); color: rgb(255 255 255 / 70%); text-underline-offset: 3px; }
	@media (max-aspect-ratio: 4/5) {
		.details { top: 35%; bottom: 17%; width: 65%; right: 7%; }
		.artwork { inset: 0; }
		.day_sun, .sunset_sun { width: 90%; left: -18%; top: -4%; }
		.day_clouds { width: 80%; left: -5%; top: 32%; }
		.sunset_clouds2 { width: 110%; left: -35%; top: 20%; }
		.sunset_clouds1 { width: 80%; left: 10%; top: 17%; }
		.night_moon { width: 45%; left: 4%; top: 5%; }
		.night_clouds { width: 100%; left: -35%; top: 16%; }
		.night_stars { width: 90%; left: 15%; top: 12%; }
	}
	@media (prefers-reduced-motion: reduce) { .next { transition: none; } }
</style>
