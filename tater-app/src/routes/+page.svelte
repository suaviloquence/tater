<script lang="ts">
	import * as pdfjsLib from 'pdfjs-dist';
	pdfjsLib.GlobalWorkerOptions.workerSrc =
		'https://cdnjs.cloudflare.com/ajax/libs/pdf.js/4.8.69/pdf.worker.mjs';

	interface Props {
		pageNumber: number;
		scale: number;
	}

	let { pageNumber = 1, scale = 1.0 } = $props() as Props;

	let canvas: HTMLCanvasElement = $state(null)!;
	let textContainer: HTMLDivElement = $state(null)!;

	async function mkPdf() {
		const pdf = await pdfjsLib.getDocument('/sample.pdf').promise;

		const page = await pdf.getPage(pageNumber);

		const canvasContext = canvas.getContext('2d')!;
		const viewport = page.getViewport({
			scale
		});

		canvas.height = viewport.height;
		canvas.width = viewport.width;

		await page.render({ canvasContext, viewport }).promise;

		const textContent = await page.getTextContent();
		const textLayer = new pdfjsLib.TextLayer({
			textContentSource: textContent,
			container: textContainer,
			viewport
		});

		await textLayer.render();
	}
</script>

<svelte:head>
	<link href="https://mozilla.github.io/pdf.js/web/viewer.css" rel="stylesheet" type="text/css" />
</svelte:head>

<div
	class="textLayer"
	id="text-container"
	bind:this={textContainer}
	style:--scale-factor={scale}
></div>
<canvas bind:this={canvas}></canvas>

{#await mkPdf()}
	Loading pdf...
{/await}

<style>
</style>
