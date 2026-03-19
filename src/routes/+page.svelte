<script lang="ts">
	import './page.css';
	import { onMount } from 'svelte';
	import * as d3 from 'd3';
	import eventsCsv from '$lib/assets/events.csv?raw';

	type EventItem = {
		id: string;
		title: string;
		tag: string;
		date: string;
		dateObject: Date;
		category: string;
		description: string;
		people: string;
		formattedDate: string;
		displayHeadline: string;
		displayDescription: string;
	};

	type PositionedEvent = EventItem & {
		top: number;
		dotY: number;
		cardHeight: number;
	};

	type LegendItem = {
		category: string;
		color: string;
	};

	type EventRow = {
		className: string;
		text: string;
	};

	const categoryPalette = ['#1d4ed8', '#0f766e', '#b45309', '#7c3aed', '#be123c', '#4d7c0f', '#0891b2'];

	let scrollY = $state(0);
	let timelineContainer: HTMLDivElement | null = $state(null);
	let timelineSvg: SVGSVGElement | null = $state(null);
	let chartWidth = $state(0);

	function parseCsv(text: string): string[][] {
		const rows: string[][] = [];
		let currentRow: string[] = [];
		let currentValue = '';
		let inQuotes = false;

		for (let index = 0; index < text.length; index += 1) {
			const character = text[index];
			const nextCharacter = text[index + 1];

			if (character === '"') {
				if (inQuotes && nextCharacter === '"') {
					currentValue += '"';
					index += 1;
				} else {
					inQuotes = !inQuotes;
				}
				continue;
			}

			if (!inQuotes && character === ',') {
				currentRow.push(currentValue.trim());
				currentValue = '';
				continue;
			}

			if (!inQuotes && character === '\n') {
				currentRow.push(currentValue.trim());
				if (currentRow.some((value) => value.length > 0)) {
					rows.push(currentRow);
				}
				currentRow = [];
				currentValue = '';
				continue;
			}

			if (character !== '\r') {
				currentValue += character;
			}
		}

		if (currentValue.length > 0 || currentRow.length > 0) {
			currentRow.push(currentValue.trim());
			if (currentRow.some((value) => value.length > 0)) {
				rows.push(currentRow);
			}
		}

		return rows;
	}

	function createDate(dateString: string): Date {
		return new Date(`${dateString}T00:00:00`);
	}

	function formatDate(value: string | Date): string {
		const date = value instanceof Date ? value : createDate(value);
		return new Intl.DateTimeFormat('en-GB', {
			day: 'numeric',
			month: 'short',
			year: 'numeric'
		}).format(date);
	}

	const [headerRow = [], ...dataRows] = parseCsv(eventsCsv);
	const headers = headerRow.map((value) => value.toLowerCase());

	const events: EventItem[] = dataRows
		.map((row) => {
			const record = Object.fromEntries(headers.map((header, index) => [header, row[index] ?? '']));
			const tag = record.tag ?? '';
			const title = record.title ?? '';
			const category = record.category ?? '';
			const dateObject = createDate(record.date ?? '');

			return {
				id: record.id ?? '',
				title,
				tag,
				date: record.date ?? '',
				dateObject,
				category,
				description: record.description ?? '',
				people: record.people ?? '',
				formattedDate: formatDate(dateObject),
				displayHeadline:
					tag && title && tag !== title
						? `${tag} - ${title}`
						: title || tag || category || 'Untitled event',
				displayDescription: (record.description ?? '').trim() || ' '
			};
		})
		.sort((first, second) => {
			const dateDelta = first.dateObject.getTime() - second.dateObject.getTime();
			return dateDelta !== 0 ? dateDelta : Number(first.id) - Number(second.id);
		});

	const categories = [...new Set(events.map((event) => event.category).filter(Boolean))];
	const categoryColor = d3
		.scaleOrdinal()
		.domain(categories)
		.range(categories.map((_, index) => categoryPalette[index % categoryPalette.length]));

	const legendItems: LegendItem[] = categories.map((category) => ({
		category,
		color: categoryColor(category)
	}));

	const startDate = events[0]?.dateObject;
	const endDate = events.at(-1)?.dateObject;

	function buildEventRows(event: EventItem): EventRow[] {
		return [
			{ className: 'event-headline', text: event.displayHeadline },
			{ className: 'event-description', text: event.displayDescription }
		];
	}

	function buildEventCard(documentRef: Document, event: EventItem, cardWidth: number): HTMLDivElement {
		const card = documentRef.createElement('div');
		card.className = 'event-card';
		card.setAttribute('xmlns', 'http://www.w3.org/1999/xhtml');
		card.style.width = `${cardWidth}px`;

		for (const row of buildEventRows(event)) {
			const element = documentRef.createElement('div');
			element.className = row.className;
			element.textContent = row.text;
			card.appendChild(element);
		}

		return card;
	}

	function measureEventHeights(cardWidth: number): number[] {
		if (!timelineContainer) {
			return events.map(() => 0);
		}

		const documentRef = timelineContainer.ownerDocument;
		const measureLayer = documentRef.createElement('div');
		measureLayer.className = 'measure-layer';
		timelineContainer.appendChild(measureLayer);

		const heights = events.map((event) => {
			const card = buildEventCard(documentRef, event, cardWidth);
			measureLayer.appendChild(card);
			return Math.ceil(card.getBoundingClientRect().height);
		});

		measureLayer.remove();
		return heights;
	}

	function renderTimeline() {
		if (!timelineSvg || !timelineContainer || events.length === 0) {
			return;
		}

		const width = Math.max(320, Math.round(timelineContainer.clientWidth));
		const compact = width < 560;
		const narrow = width < 820;
		const dateColumnWidth = compact ? 68 : narrow ? 84 : 120;
		const dotX = compact ? 86 : narrow ? 104 : 144;
		const squareSize = compact ? 8 : 10;
		const squareX = dotX + (compact ? 11 : 13);
		const textX = squareX + squareSize + (compact ? 10 : 14);
		const minCardGap = compact ? 3 : 4;
		const minTemporalGap = compact ? 1 : 2;
		const availableTextWidth = Math.max(120, width - textX - 12);
		const dotOffsetFromTop = compact ? 7 : 8;
		const eventHeights = measureEventHeights(availableTextWidth);
		const datedEvents = events.map((event, index) => ({
			...event,
			cardHeight: eventHeights[index] || 0
		}));
		const timeExtent = d3.extent(datedEvents, (event: PositionedEvent) => event.dateObject) as [Date, Date];
		const positiveGapMs = datedEvents
			.slice(1)
			.map((event, index) => event.dateObject.getTime() - datedEvents[index].dateObject.getTime())
			.filter((gapMs) => gapMs > 0);
		const smallestPositiveGapMs =
			positiveGapMs.length > 0 ? Math.min(...positiveGapMs) : 86400000;
		const totalTimeSpanMs = Math.max(0, timeExtent[1].getTime() - timeExtent[0].getTime());
		const temporalRangeHeight =
			positiveGapMs.length > 0 ? (totalTimeSpanMs / smallestPositiveGapMs) * minTemporalGap : 0;
		const topPadding = dotOffsetFromTop + minCardGap;
		const bottomPadding = minCardGap;
		const timeScale = d3
			.scaleTime()
			.domain(timeExtent)
			.range([0, temporalRangeHeight]);

		let accumulatedCardHeight = 0;

		const positionedEvents: PositionedEvent[] = datedEvents.map((event, index) => {
			const top = topPadding + accumulatedCardHeight + timeScale(event.dateObject);
			const positionedEvent = {
				...event,
				top,
				dotY: top + dotOffsetFromTop
			};

			accumulatedCardHeight += event.cardHeight;
			if (index < datedEvents.length - 1) {
				accumulatedCardHeight += minCardGap;
			}

			return positionedEvent;
		});

		const finalHeight = Math.ceil(
			(positionedEvents.at(-1)?.top ?? topPadding) + (positionedEvents.at(-1)?.cardHeight ?? 0) + bottomPadding
		);
		chartWidth = width;

		const svg = d3.select(timelineSvg);
		svg.attr('viewBox', `0 0 ${width} ${finalHeight}`).attr('width', width).attr('height', finalHeight);
		svg.selectAll('*').remove();

		svg
			.append('line')
			.attr('class', 'axis-line')
			.attr('x1', dotX)
			.attr('x2', dotX)
			.attr('y1', Math.max(0, (positionedEvents[0]?.dotY ?? topPadding) - minCardGap))
			.attr('y2', (positionedEvents.at(-1)?.dotY ?? finalHeight - bottomPadding) + minCardGap);

		const eventGroups = svg
			.selectAll('g.timeline-entry')
			.data(positionedEvents)
			.join('g')
			.attr('class', 'timeline-entry')
			.attr('transform', (event: PositionedEvent) => `translate(0, ${event.dotY})`);

		eventGroups
			.append('text')
			.attr('class', 'timeline-date')
			.attr('x', dateColumnWidth)
			.attr('y', 0)
			.attr('text-anchor', 'end')
			.attr('dominant-baseline', 'middle')
			.text((event: PositionedEvent) => event.formattedDate);

		eventGroups
			.append('circle')
			.attr('class', 'timeline-dot')
			.attr('cx', dotX)
			.attr('cy', 0)
			.attr('r', compact ? 3.5 : 4);

		eventGroups
			.append('rect')
			.attr('class', 'timeline-square')
			.attr('x', squareX)
			.attr('y', -(squareSize / 2))
			.attr('width', squareSize)
			.attr('height', squareSize)
			.attr('fill', (event: PositionedEvent) => categoryColor(event.category) ?? '#111');

		const cardGroups = eventGroups
			.append('foreignObject')
			.attr('class', 'event-foreign-object')
			.attr('x', textX)
			.attr('y', (event: PositionedEvent) => event.top - event.dotY)
			.attr('width', availableTextWidth)
			.attr('height', (event: PositionedEvent) => event.cardHeight);

		cardGroups.each(function (this: SVGForeignObjectElement, event: PositionedEvent) {
			const node = this as SVGForeignObjectElement;
			node.replaceChildren();
			node.appendChild(buildEventCard(node.ownerDocument, event, availableTextWidth));
		});
	}

	onMount(() => {
		if (!timelineContainer) {
			return;
		}

		const resizeObserver = new ResizeObserver((entries) => {
			const entry = entries[0];
			if (!entry) {
				return;
			}

			chartWidth = Math.round(entry.contentRect.width);
		});

		resizeObserver.observe(timelineContainer);
		chartWidth = Math.round(timelineContainer.clientWidth);

		return () => {
			resizeObserver.disconnect();
		};
	});

	$effect(() => {
		chartWidth;
		if (!timelineSvg || !timelineContainer) {
			return;
		}

		renderTimeline();
	});
</script>

<svelte:head>
	<title>Project Timeline</title>
	<meta name="description" content="A D3-driven project timeline generated from events.csv." />
</svelte:head>

<svelte:window bind:scrollY />

<div class="page">
	<header class:compact={scrollY > 72} class="site-header">
		<div class="header-full">
			<div class="header-copy">
				<p class="header-label">Project timeline</p>
				<h2>The UK Co-Benefits Atlas: Behind the Scene</h2>
				<p class="header-text">This is the supplementary material for VIS submission #xxxx.</p>
			</div>

			<div class="header-stats">
				<div class="stat">
					<span class="stat-label">Entries</span>
					<strong>{events.length}</strong>
				</div>
				<div class="stat">
					<span class="stat-label">Categories</span>
					<strong>{categories.length}</strong>
				</div>
				<div class="stat">
					<span class="stat-label">Duration</span>
					<strong>{startDate ? formatDate(startDate) : ''} to {endDate ? formatDate(endDate) : ''}</strong>
				</div>
			</div>
		</div>

		<div class="header-compact">
			<strong>Project timeline</strong>
			<span>{events.length} events</span>
			<span>{startDate ? formatDate(startDate) : ''} to {endDate ? formatDate(endDate) : ''}</span>
		</div>
	</header>

	{#if legendItems.length > 0}
		<section class="legend" aria-label="Category legend">
			{#each legendItems as item}
				<div class="legend-item">
					<span class="legend-swatch" style={`background-color: ${item.color}`}></span>
					<span>{item.category}</span>
				</div>
			{/each}
		</section>
	{/if}

	<div class="timeline-canvas" bind:this={timelineContainer}>
		<svg bind:this={timelineSvg} class="timeline-svg" aria-label="Project events timeline"></svg>
	</div>
</div>
