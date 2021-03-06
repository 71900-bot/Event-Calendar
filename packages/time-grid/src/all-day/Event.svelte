<script>
	import {getContext, onMount, afterUpdate, tick, createEventDispatcher} from 'svelte';
	import {is_function} from 'svelte/internal';
	import {
		createEventContent,
		height,
		toEventWithLocalDates,
		toViewWithLocalDates,
		setContent
	} from '@event-calendar/common';

	export let chunk;
	export let longChunks = {};

	let {displayEventEnd, eventBackgroundColor, eventClick, eventColor, eventContent, eventDidMount,
		eventMouseEnter, eventMouseLeave, theme, _view, _intlEventTime, _interaction, _classes, _draggable} = getContext('state');
	let {_viewResources} = getContext('view-state');

	const dispatch = createEventDispatcher();

	let el;
	let event;
	let classes;
	let style;
	let content;
	let timeText;
	let margin = 1;
	let hidden = false;
	let display;

	$: event = chunk.event;

	$: {
		display = event.display;

		// Class & Style
		let bgColor = event.backgroundColor || $eventBackgroundColor || $eventColor;
		style =
			`width:calc(${chunk.days * 100}% + ${(chunk.days - 1) * 7}px);` +
			`margin-top:${margin}px;`
		;
		if (bgColor) {
			style += `background-color:${bgColor};`;
		}

		classes = $_classes($theme.event, event);
	}

	// Content
	$: [timeText, content] = createEventContent(chunk, $displayEventEnd, $eventContent, $theme, $_intlEventTime, $_view);

	onMount(() => {
		if (is_function($eventDidMount)) {
			$eventDidMount({
				event: toEventWithLocalDates(event),
				timeText,
				el,
				view: toViewWithLocalDates($_view)
			});
		}
	});

	afterUpdate(reposition);

	function createHandler(fn, display) {
		return display !== 'preview' && is_function(fn)
			? jsEvent => fn({event: toEventWithLocalDates(event), el, jsEvent, view: toViewWithLocalDates($_view)})
			: undefined;
	}

	function createDragHandler(resize) {
		return jsEvent => $_interaction.drag.startTimeGrid(event, el, jsEvent, _viewResources, true, resize);
	}

	function reposition() {
		if (!el || display === 'preview') {
			return;
		}
		chunk.top = 0;
		if (chunk.prev) {
			if (chunk.prev.bottom === undefined) {
				// 'prev' is not ready yet, try again later
				tick().then(reposition);
				return;
			}
			chunk.top = chunk.prev.bottom + 1;
		}
		chunk.bottom = chunk.top + height(el);
		let m = 1;
		let key = chunk.date.getTime();
		if (longChunks[key]) {
			for (let longChunk of longChunks[key]) {
				if (longChunk.bottom === undefined) {
					// 'longChunk' is not ready yet, try again later
					tick().then(reposition);
					return;
				}
				if (chunk.top < longChunk.bottom && chunk.bottom > longChunk.top) {
					let offset = longChunk.bottom - chunk.top + 1;
					m += offset;
					chunk.top += offset;
					chunk.bottom += offset;
				}
			}
		}
		margin = m;
	}
</script>

<div
	bind:this={el}
	class="{classes}"
	{style}
	on:click={createHandler($eventClick, display)}
	on:mouseenter={createHandler($eventMouseEnter, display)}
	on:mouseleave={createHandler($eventMouseLeave, display)}
	on:pointerdown={display === 'auto' && $_draggable(event) ? createDragHandler() : undefined}
>
	<div class="{$theme.eventBody}" use:setContent={content}></div>
	<svelte:component
		this={$_interaction.resizer}
		{event}
		on:pointerdown={createDragHandler(true)}
	/>
</div>

<svelte:window on:resize={reposition}/>