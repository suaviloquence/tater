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

	interface SelectedObject {
		text: string;
		range: Range;
	}

	let selection: SelectedObject | null = $state(null);

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

	/**
	 * heuristic to join nodes in a text layer to a string with the right whitespace

	 * if two following lines have a different font size, left indent, or (for
		third+ lines) has a different line spacing, assume it's a new paragraph,
		otherwise it's a space.
	 * assumes the given range is made of pdf.js-generated text nodes.  specifically,
		1. `endContainer` is a subsequent sibling of `startContainer`
		2. text is contained exclusively in sibling `<span>`s that have
		   the `left` and `top` css properties inline
	* if `joinHyphens` is true, the program will consider any hyphens on line breaks
	   that are not new paragraphs to be joining the same word, and will remove it.

	**/
	function textRangeToString(range: Range, joinHyphens: boolean = true) {
		if (range.startContainer === range.endContainer) return range.toString();

		// assumes these both only have one text node
		const startSpan = range.startContainer.parentElement!;
		let out = '';
		let first = true;

		interface ParagraphHeuristic {
			left: number;
			top: number;
			lineSpacing: number | null;
			fontSize: string;
		}

		const scale = Number.parseFloat(startSpan.style.transform.match(/scaleX\(([\d\.]+)\)/)?.at(1)!);

		let heuristic: ParagraphHeuristic = {
			left: Number.parseFloat(startSpan.style.left) * scale,
			top: Number.parseFloat(startSpan.style.top),
			lineSpacing: null,
			fontSize: startSpan.style.fontSize
		};

		function fuzzyeq(a: number, b: number): boolean {
			return Math.abs(a - b) < 0.1;
		}

		// TODO: refine heuristic for indents
		function isParagraph(
			element: HTMLElement,
			prev: ParagraphHeuristic
		): [boolean, ParagraphHeuristic] {
			const scale = Number.parseFloat(
				element.style.transform.match(/scaleX\(([\d\.]+)\);/)?.at(1) ?? '1'
			);
			let top = Number.parseFloat(element.style.top) / scale;
			let curr: ParagraphHeuristic = {
				left: Number.parseFloat(element.style.left) / scale,
				top,
				lineSpacing: top - prev.top,
				fontSize: element.style.fontSize
			};

			// same line, no para
			return [
				!fuzzyeq(prev.left, curr.left) ||
					(prev.lineSpacing && curr.lineSpacing && !fuzzyeq(prev.lineSpacing, curr.lineSpacing)) ||
					prev.fontSize !== curr.fontSize,
				curr
			];
		}

		for (const node of range.cloneContents().children) {
			if (!(node instanceof HTMLSpanElement)) continue;
			// is this a paragraph? heuristic
			if (first) first = false;
			else {
				let [para, heur] = isParagraph(node, heuristic);
				if (para) {
					out += '\n';
				} else {
					if (/-|—­|–/.test(out.charAt(out.length - 1))) {
						if (joinHyphens) out = out.substring(0, out.length - 1);
						// otherwise, just don't add a space
					} else {
						out += ' ';
					}
				}

				heuristic = heur;
			}

			out += node.textContent;
		}

		return out;
	}

	document.addEventListener('selectionchange', () => (selection = handleSelection()));

	function handleSelection(): SelectedObject | null {
		const selection = document.getSelection();
		if (!selection || selection.rangeCount < 1) return null;
		const range = selection.getRangeAt(0);
		if (!range || range.collapsed) return null;
		if (
			!textContainer.contains(range.startContainer) ||
			!textContainer.contains(range.endContainer)
		)
			return null;

		const text = textRangeToString(range);

		return { text, range };
	}

	function offsetFor(range: Range): { left: string; top: string } {
		let span = range.startContainer.parentElement as HTMLSpanElement;

		const scale = Number.parseFloat(
			span.style.transform.match(/scaleX\(([\d\.]+)\);/)?.at(1) ?? '1'
		);
		let textLeft = Number.parseFloat(span.style.left) * scale;
		let textTop = Number.parseFloat(span.style.top);

		const top = -1;
		const left = 0;

		return { left: `calc(${textLeft}% + ${left}em)`, top: `calc(${textTop}% + ${top}em)` };
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
{:then}
	{#if selection}
		<div
			class="tooltip absolute"
			style:left={offsetFor(selection.range).left}
			style:top={offsetFor(selection.range).top}
		>
			<button aria-label="create annotation">+</button>
		</div>
	{/if}
{/await}

<style>
</style>
