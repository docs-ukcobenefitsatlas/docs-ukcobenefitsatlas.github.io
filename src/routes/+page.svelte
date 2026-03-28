<script lang="ts">
	import './page.css';
	import { onMount } from 'svelte';
	import * as d3 from 'd3';
	import MarkdownDrawer from '$lib/components/MarkdownDrawer.svelte';
	import eventsCsv from '$lib/assets/events.csv?raw';

	type EventItem = {
		id: string;
		title: string;
		tag: string;
		date: string;
		dateObject: Date;
		category: string;
		code: string;
		file: string;
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
		label: string;
		color: string;
	};

	type EventRow = {
		className: string;
		text: string;
	};

	type CategoryColumn = {
		key: string;
		label: string;
		color: string;
	};

	const categoryColumns: CategoryColumn[] = [
		{ key: 'Atlas Discourse', label: 'Atlas discourse', color: '#1d4ed8' },
		{ key: 'Design Workshop', label: 'Workshop', color: '#0f766e' },
		{ key: 'Prototype & Implementation', label: 'Prototype', color: '#b45309' },
		{ key: 'Stakeholder Engagement', label: 'Stakeholder engagement', color: '#7c3aed' }
	];

	let scrollY = $state(0);
	let headerElement: HTMLElement | null = $state(null);
	let timelineContainer: HTMLDivElement | null = $state(null);
	let timelineSvg: SVGSVGElement | null = $state(null);
	let chartWidth = $state(0);
	let drawerTop = $state(96);
	let activeEvent: EventItem | null = $state(null);
	let targetCode = $state('');

	function scrollToCode(code: string) {
		if (!code || !timelineSvg) return;
		const el = timelineSvg.querySelector(`[data-code="${CSS.escape(code)}"]`);
		if (!el) return;
		el.scrollIntoView({ behavior: 'smooth', block: 'center' });
		const event = events.find((e) => e.code === code);
		activeEvent = event?.file ? event : null;
	}

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
		// Support both DD/MM/YYYY and YYYY-MM-DD formats
		if (/^\d{2}\/\d{2}\/\d{4}$/.test(dateString)) {
			const [day, month, year] = dateString.split('/');
			return new Date(`${year}-${month}-${day}T00:00:00`);
		}
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
				code: record.code ?? '',
				file: record.file ?? '',
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

	const categoryColor = d3
		.scaleOrdinal()
		.domain(categoryColumns.map((column) => column.key))
		.range(categoryColumns.map((column) => column.color));

	const legendItems: LegendItem[] = categoryColumns.map((column) => ({
		category: column.key,
		label: column.label,
		color: column.color
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
		card.className = event.file ? 'event-card has-file' : 'event-card';
		card.setAttribute('xmlns', 'http://www.w3.org/1999/xhtml');
		card.style.width = `${cardWidth}px`;
		if (event.file) {
			card.title = event.file;
			card.addEventListener('click', () => {
				activeEvent = event;
			});
		}

		for (const [index, row] of buildEventRows(event).entries()) {
			const element = documentRef.createElement('div');
			element.className = row.className;
			if (index === 0 && event.file) {
				const text = documentRef.createElement('span');
				text.className = 'event-headline-text';
				text.textContent = row.text;

				const icon = documentRef.createElement('span');
				icon.className = 'event-file-icon';
				icon.setAttribute('aria-hidden', 'true');

				element.appendChild(text);
				element.appendChild(icon);
			} else {
				element.textContent = row.text;
			}
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
		const squareGap = compact ? 3 : 4;
		const markerGridX = dotX + (compact ? 11 : 13);
		const markerGridWidth = categoryColumns.length * squareSize + (categoryColumns.length - 1) * squareGap;
		const textX = markerGridX + markerGridWidth + (compact ? 10 : 14);
		const minCardGap = compact ? 3 : 4;
		const minTemporalGap = compact ? 1 : 2;
		const availableTextWidth = Math.max(120, width - textX - 12);
		// const cardWidth = Math.min(availableTextWidth, compact ? 220 : narrow ? 300 : 360);
		const cardWidth = Math.min(availableTextWidth, compact ? 350 : narrow ? 400 : 450);

		const dotOffsetFromTop = compact ? 7 : 8;
		const eventHeights = measureEventHeights(cardWidth);
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
			.attr('data-code', (event: PositionedEvent) => event.code)
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

		const markerGroups = eventGroups
			.append('g')
			.attr('class', 'timeline-marker-grid')
			.attr('transform', `translate(${markerGridX}, ${-(squareSize / 2)})`);

		markerGroups.each(function (this: SVGGElement, event: PositionedEvent) {
			const group = d3.select(this);

			group
				.selectAll('rect.timeline-square')
				.data(categoryColumns)
				.join('rect')
				.attr('class', 'timeline-square')
				.attr('x', (_: CategoryColumn, index: number) => index * (squareSize + squareGap))
				.attr('y', 0)
				.attr('width', squareSize)
				.attr('height', squareSize)
				.attr('fill', (column: CategoryColumn) =>
					column.key === event.category ? categoryColor(column.key) : '#fff'
				)
				.attr('stroke', (column: CategoryColumn) =>
					column.key === event.category ? categoryColor(column.key) : '#d9d9d9'
				);
		});

		const cardGroups = eventGroups
			.append('foreignObject')
			.attr('class', 'event-foreign-object')
			.attr('x', textX)
			.attr('y', (event: PositionedEvent) => event.top - event.dotY)
			.attr('width', cardWidth)
			.attr('height', (event: PositionedEvent) => event.cardHeight);

		cardGroups.each(function (this: SVGForeignObjectElement, event: PositionedEvent) {
			const node = this as SVGForeignObjectElement;
			node.replaceChildren();
			node.appendChild(buildEventCard(node.ownerDocument, event, cardWidth));
		});

		if (targetCode) {
			const code = targetCode;
			requestAnimationFrame(() => scrollToCode(code));
		}
	}

	onMount(() => {
		const observers: ResizeObserver[] = [];

		if (timelineContainer) {
			const timelineObserver = new ResizeObserver((entries) => {
				const entry = entries[0];
				if (!entry) {
					return;
				}

				chartWidth = Math.round(entry.contentRect.width);
			});

			timelineObserver.observe(timelineContainer);
			chartWidth = Math.round(timelineContainer.clientWidth);
			observers.push(timelineObserver);
		}

		const initialHash = window.location.hash.slice(1);
		if (initialHash) targetCode = initialHash;

		const handleHashChange = () => {
			targetCode = window.location.hash.slice(1);
			requestAnimationFrame(() => scrollToCode(targetCode));
		};
		window.addEventListener('hashchange', handleHashChange);

		if (headerElement) {
			/*
			const headerObserver = new ResizeObserver(() => {
				if (!headerElement) {
					return;
				}

				drawerTop = Math.ceil(headerElement.getBoundingClientRect().height) + 12;
			});

			headerObserver.observe(headerElement);
			drawerTop = Math.ceil(headerElement.getBoundingClientRect().height) + 12;
			observers.push(headerObserver);
			*/
		}

		return () => {
			for (const observer of observers) {
				observer.disconnect();
			}
			window.removeEventListener('hashchange', handleHashChange);
		};
	});

	$effect(() => {
		chartWidth;
		if (!timelineSvg || !timelineContainer) {
			return;
		}

		renderTimeline();
	});

	$effect(() => {
		if (!timelineSvg) return;
		timelineSvg.querySelectorAll('.event-card.is-active').forEach((el) => el.classList.remove('is-active'));
		if (activeEvent) {
			const group = timelineSvg.querySelector(`[data-code="${CSS.escape(activeEvent.code)}"]`);
			group?.querySelector('.event-card')?.classList.add('is-active');
		}
	});
</script>

<svelte:head>
	<title>Project Timeline</title>
	<meta name="description" content="Project supplementary material." />
</svelte:head>

<svelte:window bind:scrollY />

<div class="page">
	<header bind:this={headerElement} class:compact={scrollY > 72} class="site-header">
		<div class="header-full">
			<div class="header-copy">
				<p class="header-label">Project timeline</p>
				<h2>The UK Co-Benefits Atlas: Behind the Scene</h2>
				<p class="header-text">This is the supplementary material for VIS submission #1678. The timeline records four strands of design related activities for the UK Co-Benefits Atlas to contextualize rationales of design decisions. It does not include discussions for writing the paper.</p>
			</div>

			<div class="header-stats">
				<div class="stat">
					<span class="stat-label">Entries</span>
					<strong>{events.length}</strong>
				</div>
				<!-- <div class="stat">
					<span class="stat-label">Categories</span>
					<strong>{legendItems.length}</strong>
				</div> -->
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
					<span>{item.label}</span>
				</div>
			{/each}
		</section>
	{/if}

	<div class:has-drawer={Boolean(activeEvent)} class="timeline-layout" style={`--drawer-top: ${drawerTop}px;`}>
		<div class="timeline-canvas" bind:this={timelineContainer}>
			<svg bind:this={timelineSvg} class="timeline-svg" aria-label="Project events timeline"></svg>
		</div>

		<MarkdownDrawer
			open={Boolean(activeEvent)}
			fileKey={activeEvent?.file ?? ''}
			title={activeEvent?.displayHeadline ?? ''}
			onClose={() => {
				activeEvent = null;
			}}
		/>
	</div>
</div>
