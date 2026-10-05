<script lang="ts">
	import { onMount, untrack } from 'svelte';

	let { text, reducedMotion = false }: { text: string; reducedMotion?: boolean } = $props();
	let visible = $state('');
	let sizingText = $state('');
	let animating = $state(false);
	let content: HTMLDivElement;
	let measurement: HTMLParagraphElement;
	let fontSize = $state(18);

	function fitText() {
		if (!content || !measurement) return;
		const preferred = Math.min(content.clientWidth * 0.055, window.innerHeight * 0.025, 22);
		let low = 1;
		let high = preferred;
		// Measure the complete message so its font stays stable as characters appear.
		for (let attempt = 0; attempt < 12; attempt++) {
			const candidate = (low + high) / 2;
			measurement.style.fontSize = `${candidate}px`;
			if (measurement.scrollHeight <= content.clientHeight && measurement.scrollWidth <= content.clientWidth) low = candidate;
			else high = candidate;
		}
		fontSize = low;
		measurement.style.fontSize = `${fontSize}px`;
	}

	$effect(() => {
		const target = text;
		const skipAnimation = reducedMotion;
		const previous = Array.from(untrack(() => visible));
		const characters = Array.from(target);
		// Reserve room for the longer message while the outgoing text erases.
		sizingText = !skipAnimation && previous.length > characters.length ? previous.join('') : target;
		let frame = 0;
		let cancelled = false;
		if (skipAnimation) {
			visible = target;
			animating = false;
		} else {
			const eraseDuration = previous.length ? Math.min(900, previous.length * 5) : 0;
			const typingStarts = eraseDuration + (previous.length ? 150 : 0);
			const start = performance.now();
			animating = true;
			function tick(now: number) {
				const elapsed = now - start;
				if (elapsed < eraseDuration) {
					visible = previous.slice(0, Math.ceil(previous.length * (1 - elapsed / eraseDuration))).join('');
				} else {
					const count = Math.max(0, Math.floor((elapsed - typingStarts) / 8));
					visible = characters.slice(0, count).join('');
					if (count >= characters.length) {
						animating = false;
						return;
					}
				}
				frame = requestAnimationFrame(tick);
			}
			frame = requestAnimationFrame(tick);
		}
		// Wait for the full measurement text to update before sizing it.
		queueMicrotask(() => { if (!cancelled) fitText(); });
		return () => {
			cancelled = true;
			cancelAnimationFrame(frame);
		};
	});

	onMount(() => {
		const observer = new ResizeObserver(fitText);
		observer.observe(content);
		return () => observer.disconnect();
	});
</script>

<div class="message">
	<p class="accessible" aria-live="polite" aria-atomic="true">{text}</p>
	<div class="content" bind:this={content} style:font-size={`${fontSize}px`}>
		<p class="measurement" bind:this={measurement} aria-hidden="true">{sizingText}<span class="caret-space"></span></p>
		<p class="typed" aria-hidden="true">{visible}<span class="caret" class:animating></span></p>
	</div>
</div>

<style>
	.message { width: 100%; height: 100%; padding: min(5cqi, 2dvh, 24px); }
	.content { position: relative; width: 100%; height: 100%; line-height: 1.5; overflow-wrap: anywhere; }
	p { margin: 0; white-space: pre-wrap; }
	.measurement { visibility: hidden; pointer-events: none; }
	.typed { position: absolute; inset: 0; }
	.caret, .caret-space { display: inline-block; width: 0.15em; height: 1em; margin-left: 0.15em; vertical-align: -0.12em; }
	.caret { background: currentColor; opacity: 0; }
	.caret.animating { animation: blink 800ms step-end infinite; }
	.accessible { position: absolute; width: 1px; height: 1px; padding: 0; margin: -1px; overflow: hidden; clip-path: inset(50%); white-space: nowrap; }
	@keyframes blink { 0%, 100% { opacity: 1; } 50% { opacity: 0; } }
	@media (prefers-reduced-motion: reduce) { .caret.animating { animation: none; } }
</style>
