<script context="module" lang="ts">
	import { scrollSpeed as __scrollSpeed, scrollStop as _scrollStop } from "$lib/utils";

	const scrollStop = _scrollStop();
	const _scrollSpeed = __scrollSpeed();

	const getIndexes = (
		itemCount: number,
		itemSize: number,
		size: number,
		overScan: number,
		scrollPosition: number
	) => {
		const indexes: number[] = [];

		const startIndexTemp = ~~(scrollPosition / itemSize);
		const startIndexOverScan = startIndexTemp > overScan ? startIndexTemp - overScan : 0;
		const startIndex = startIndexOverScan >= 0 ? startIndexOverScan : startIndexTemp;

		const endIndexTemp = Math.min(itemCount, ~~((scrollPosition + size) / itemSize));
		const endIndexOverScan = endIndexTemp + overScan;
		const endIndex = endIndexOverScan < itemCount ? endIndexOverScan : itemCount;

		for (let i = startIndex; i < endIndex; i++) indexes.push(i);

		return indexes;
	};
</script>

<script lang="ts">
	import type { ScrollBehavior } from "$lib/types";

	export let itemCount: number;
	export let itemSize: number;
	export let height: number | string = "100%";
	export let width = "100%";

	export let stickyIndices: number[] = [];

	export let overScan = 1;

	export let marginLeft = 0;
	export let marginTop = 0;
	export let layout: "vertical" | "horizontal" = "vertical";

	export let scrollPosition = 0;
	export let scrollBehavior: ScrollBehavior = "auto";

	export let getKey: (index: number) => unknown = (index: number) => index;

	let list: HTMLElement;
	let _scrollPosition = scrollPosition;
	let headerHeight = 0;
	let offsetHeight = 0;
	let clientHeight = 0;
	let offsetWidth = 0;
	let clientWidth = 0;
	let indexes: number[] = [];

	let manualScroll = false;
	let isScrolling = false;
	let isScrollingFast = false;

	export const scrollToIndex = (index: number, behavior: ScrollBehavior = scrollBehavior) => {
		scrollTo(index * itemSize + (isVertical ? marginTop : marginLeft) + headerHeight, behavior);
	};

	export const scrollToPosition = (
		position: number,
		behavior: ScrollBehavior = scrollBehavior
	) => {
		scrollTo(position, behavior);
	};

	const scrollTo = (direction: number, behavior: ScrollBehavior = scrollBehavior) => {
		if (list) {
			manualScroll = true;

			list.scrollTo({ [isVertical ? "top" : "left"]: direction, behavior });

			manualScroll = false;
		}
	};

	const scrollToManual = (scrollPosition: number) => {
		if (!manualScroll && !isScrolling) {
			manualScroll = true;

			list.scrollTo({ top: scrollPosition, behavior: scrollBehavior });

			manualScroll = false;
		}
	};

	const getItemStyle = (index: number) => {
		const ixis = index * itemSize;

		return `position: absolute; transform: translate3d(${
			isVertical
				? `${marginLeft}px, ${ixis + marginTop}px`
				: `${ixis + marginLeft}px, ${marginTop}px`
		}, 0px); ${itemSizeInternal} will-change: transform;`;
	};

	const onScroll = ({ currentTarget }: { currentTarget: HTMLDivElement }) => {
		isScrolling = true;

		if (!manualScroll) {
			const direction = currentTarget[isVertical ? "scrollTop" : "scrollLeft"];

			_scrollPosition = Math.max(0, direction - headerHeight);
			scrollPosition = direction;

			scrollSpeed(_scrollPosition);
		}

		scrollStop(() => {
			isScrolling = false;
		});
	};

	$: isVertical = layout === "vertical";

	$: innerSize = itemCount * itemSize;

	$: itemSizeInternal = isVertical
		? `height: ${itemSize}px; width: ${
				marginLeft > 0 ? `${clientWidth - marginLeft}px` : "100%"
		  };`
		: `height: ${
				marginTop > 0 ? `${clientHeight - marginTop}px` : "100%"
		  }; width: ${itemSize}px;`;

	$: size = isVertical ? offsetHeight : offsetWidth;

	$: if (size) {
		indexes = getIndexes(itemCount, itemSize, size, overScan, _scrollPosition);
	}

	$: if (list) {
		scrollToManual(scrollPosition);
	}

	$: scrollSpeed = _scrollSpeed(size, {
		fast: () => {
			isScrollingFast = true;
		},
		slow: () => {
			isScrollingFast = false;
		},
	});
</script>

<div
	style:position="relative"
	style:overflow="auto"
	style:height={typeof height === "number" ? `${height}px` : height}
	style:width
	on:scroll={onScroll}
	bind:this={list}
	bind:offsetHeight
	bind:clientHeight
	bind:offsetWidth
	bind:clientWidth
>
	{#if $$slots.header}
		<div bind:offsetHeight={headerHeight}>
			<slot name="header" />
		</div>
	{/if}

	<div
		style:height={isVertical ? `${innerSize}px` : "100%"}
		style:width={!isVertical ? `${innerSize}px` : "100%"}
	>
		{#if stickyIndices.length && indexes.length}
			{@const stickyIndex = Math.max(...stickyIndices.filter((i) => i < indexes[0]))}
			{#if stickyIndex >= 0}
				<div
					style:position="sticky"
					style:top={isVertical ? `${marginTop}px` : "0px"}
					style:left={isVertical ? "0px" : `${marginLeft}px`}
					style:z-index="1"
				>
					<slot name="item" index={stickyIndex}>Missing template</slot>
				</div>
			{/if}
		{/if}

		{#each indexes as index (getKey(index))}
			{@const style = getItemStyle(index)}

			{#if !isScrollingFast || !$$slots.placeholder}
				<slot name="item" {index} {style}>Missing template</slot>
			{:else}
				<slot name="placeholder" {index} {style}>Missing placeholder</slot>
			{/if}
		{/each}
	</div>

	<slot name="footer" />
</div>
