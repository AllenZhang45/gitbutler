<script lang="ts" module>
	import { type LineSelectionParams } from '$lib/hunkDiff/lineSelection.svelte';
	export type LineClickParams = LineSelectionParams;
</script>

<script lang="ts">
	import Button from '$lib/Button.svelte';
	import Checkbox from '$lib/Checkbox.svelte';
	import Icon from '$lib/Icon.svelte';
	import HunkDiffBody from '$lib/hunkDiff/HunkDiffBody.svelte';
	import {
		type ContentSection,
		type LineId,
		type LineSelector,
		parseHunk
	} from '$lib/utils/diffParsing';
	import type { ContextMenuParams } from '$lib/hunkDiff/HunkDiffRow.svelte';
	interface Props {
		filePath: string;
		hunkStr: string;
		tabSize?: number;
		wrapText?: boolean;
		diffFont?: string;
		diffLigatures?: boolean;
		inlineUnifiedDiffs?: boolean;
		diffContrast?: 'light' | 'medium' | 'strong';
		staged?: boolean;
		stagedLines?: LineId[];
		hideCheckboxes?: boolean;
		selectedLines?: LineSelector[];
		isHidden?: boolean;
		draggingDisabled?: boolean;
		whyHidden?: string;
		onShowDiffClick?: () => void;
		onChangeStage?: (staged: boolean) => void;
		onLineClick?: (params: LineSelectionParams) => void;
		clearLineSelection?: (fileName: string) => void;
		onQuoteSelection?: () => void;
		onCopySelection?: (contentSections: ContentSection[]) => void;
		handleLineContextMenu?: (params: ContextMenuParams) => void;
		clickOutsideExcludeElement?: HTMLElement;
	}

	const {
		filePath,
		hunkStr,
		tabSize = 4,
		wrapText = true,
		diffFont = 'var(--fontfamily-mono)',
		diffLigatures = true,
		diffContrast = 'medium',
		inlineUnifiedDiffs = false,
		staged,
		stagedLines,
		hideCheckboxes,
		selectedLines,
		isHidden,
		whyHidden,
		onShowDiffClick,
		onChangeStage,
		onLineClick,
		clearLineSelection,
		onCopySelection,
		onQuoteSelection,
		handleLineContextMenu,
		clickOutsideExcludeElement,
		draggingDisabled
	}: Props = $props();

	const BORDER_WIDTH = 1;

	let tableWidth = $state<number>(0);
	let tableHeight = $state<number>(0);
	let numberHeaderWidth = $state<number>(0);

	const hunk = $derived(parseHunk(hunkStr));

	function handleCopySelection() {
		onCopySelection?.(hunk.contentSections);
	}

	const hunkSummary = $derived(
		`@@ -${hunk.oldStart},${hunk.oldLines} +${hunk.newStart},${hunk.newLines} @@`
	);
	const showingCheckboxes = $derived(!hideCheckboxes && staged !== undefined);
	const colspan = $derived(showingCheckboxes ? 3 : 2);
</script>

<div
	bind:clientWidth={tableWidth}
	bind:clientHeight={tableHeight}
	class="table__wrapper hide-native-scrollbar contrast-{diffContrast}"
	style="--tab-size: {tabSize}; --diff-font: {diffFont};"
	style:font-variant-ligatures={diffLigatures ? 'common-ligatures' : 'none'}
>
	<table class="table__section">
		<thead class="table__title" class:draggable={!draggingDisabled}>
			<tr>
				<th
					bind:clientWidth={numberHeaderWidth}
					class="table__checkbox-container"
					style="--border-width: {BORDER_WIDTH}px;"
					class:stageable={showingCheckboxes}
					class:staged={showingCheckboxes && staged}
					{colspan}
					onclick={() => {
						if (showingCheckboxes) {
							onChangeStage?.(!staged);
						}
					}}
				>
					<div class="table__checkbox">
						{#if staged && !hideCheckboxes}
							<Checkbox checked={staged} small style="ghost" />
						{:else if showingCheckboxes}
							<div class="table__checkbox-unstaged">
								<Icon name="minus-small" />
							</div>
						{/if}
					</div>

					<div
						class="table__title-content"
						style="--number-col-width: {numberHeaderWidth}px; --table-width: {tableWidth}px; --border-width: {BORDER_WIDTH}px; --top: -{BORDER_WIDTH}px"
					>
						<span>
							{hunkSummary}
						</span>
					</div>
				</th>
			</tr>
		</thead>

		{#if isHidden}
			<tbody class="table__hiddenRows">
				<tr>
					<td class="table__hiddenRows__count"></td>
					<td class="table__hiddenRows__count"></td>
					<td>
						<div class="table__hiddenRows__content">
							<p class="text-12 table__hiddenRows__caption">
								{#if whyHidden}
									{whyHidden}
								{:else}
									Diff is too large to display
								{/if}
							</p>
							<Button kind="outline" onclick={onShowDiffClick} icon="eye-shown">Show anyway</Button>
						</div>
					</td>
				</tr>
			</tbody>
		{:else}
			<HunkDiffBody
				comment={hunk.comment}
				{filePath}
				content={hunk.contentSections}
				{onLineClick}
				clearLineSelection={() => clearLineSelection?.(filePath)}
				{wrapText}
				{tabSize}
				{inlineUnifiedDiffs}
				{selectedLines}
				{numberHeaderWidth}
				onCopySelection={onCopySelection && handleCopySelection}
				{onQuoteSelection}
				{staged}
				{stagedLines}
				{hideCheckboxes}
				{handleLineContextMenu}
				{clickOutsideExcludeElement}
			/>
		{/if}
	</table>
</div>

<style lang="postcss">
	.table__wrapper {
		border-radius: var(--radius-m);
		background-color: var(--clr-diff-line-bg);
		border: 1px solid var(--clr-border-2);
		overflow: hidden;
		width: 100%;
	}

	table,
	.table__section {
		width: 100%;
		font-family: var(--diff-font);
		border-collapse: separate;
		border-spacing: 0;
		user-select: none;
	}

	thead {
		width: 100%;
		padding: 0;
	}

	th,
	tr {
		padding: 0;
		margin: 0;
	}

	table thead th {
		top: 0;
		left: 0;
		position: sticky;
		height: 28px;
	}

	.table__checkbox-container {
		border-right: 1px solid var(--clr-border-2);
		border-bottom: 1px solid var(--clr-border-2);
		background-color: var(--clr-diff-count-bg);
		box-sizing: border-box;

		&.stageable {
			cursor: pointer;
		}

		&.staged {
			background-color: var(--clr-diff-selected-count-bg);
			border-color: var(--clr-diff-selected-count-border);
		}
	}

	.table__checkbox {
		padding: 4px;
		display: flex;
		justify-content: flex-start;
		align-items: center;
		pointer-events: none;
	}

	.table__checkbox-unstaged {
		display: flex;
		justify-content: center;
		align-items: center;

		color: var(--clr-diff-count-checkmark);
		margin: 0;
		padding: 0;
		width: 16px;
		height: 16px;
	}

	.table__title {
		user-select: none;
	}

	.draggable {
		cursor: grab;
	}

	.table__drag-handle {
		position: fixed;
		right: 6px;
		top: 6px;
		box-sizing: border-box;
		background-color: var(--clr-bg-1);
		display: flex;
		justify-content: center;
		align-items: center;
		border-radius: var(--radius-m);
		opacity: 0;
		transform: scale(0.9);
		transform-origin: top right;
		pointer-events: none;
		color: var(--clr-text-2);
		transition:
			opacity 0.2s,
			transform 0.2s;
	}

	.table__lock {
		position: fixed;
		right: 6px;
		top: 6px;
		box-sizing: border-box;
		background-color: var(--clr-theme-warn-soft);
		display: flex;
		justify-content: center;
		align-items: center;
		border-radius: var(--radius-m);
		pointer-events: none;
		color: var(--clr-text-2);
		transition: transform var(--transition-medium);
	}

	.table__title-content {
		color: var(--clr-text-2);
		font-size: 12px;

		position: absolute;
		top: var(--top);
		left: var(--number-col-width);
		width: calc(var(--table-width) - var(--number-col-width));
		height: calc(100% + var(--border-width) * 2);
		box-sizing: border-box;
		padding: 4px 6px;
		text-wrap: nowrap;

		display: flex;
		align-items: center;
		border-bottom: 1px solid var(--clr-border-2);
		border-top-right-radius: var(--radius-m);
	}

	/* HIDDINE LINES STATE */

	.table__hidden-rows {
		display: none;
	}

	.table__hiddenRows__count {
		width: 24px;
		background-color: var(--clr-diff-count-bg);

		&:nth-child(2) {
			border-right: 1px solid var(--clr-border-2);
		}
	}

	.table__hiddenRows__content {
		display: flex;
		align-items: center;
		justify-content: center;
		flex-direction: column;
		gap: 14px;
		padding: 40px 24px;
		background-color: var(--clr-bg-1-muted);
	}

	.table__hiddenRows__caption {
		color: var(--clr-text-2);
		text-align: center;
	}

	/* CONTRAST MODIFIERS */

	.contrast-light {
		--clr-diff-count-text: var('--', var(--clr-diff-count-text));
		/* deletion */
		--clr-diff-deletion-line-bg: var('--', var(--clr-diff-deletion-line-bg));
		--clr-diff-deletion-line-highlight: var('--', var(--clr-diff-deletion-line-highlight));
		--clr-diff-deletion-count-bg: var('--', var(--clr-diff-deletion-count-bg));
		--clr-diff-deletion-count-text: var('--', var(--clr-diff-deletion-count-text));
		--clr-diff-deletion-count-border: var('--', var(--clr-diff-deletion-count-border));
		/* addition */
		--ctx-diff-addition-line-bg: var('--', var(--clr-diff-addition-line-bg));
		--clr-diff-addition-line-highlight: var('--', var(--clr-diff-addition-line-highlight));
		--clr-diff-addition-count-bg: var('--', var(--clr-diff-addition-count-bg));
		--clr-diff-addition-count-text: var('--', var(--clr-diff-addition-count-text));
		--clr-diff-addition-count-border: var('--', var(--clr-diff-addition-count-border));
	}

	.contrast-medium {
		--clr-diff-count-text: var(--clr-diff-count-text-contrast-2);
		/* deletion */
		--clr-diff-deletion-line-bg: var(--clr-diff-deletion-contrast-2-line-bg);
		--clr-diff-deletion-line-highlight: var(--clr-diff-deletion-contrast-2-line-highlight);
		--clr-diff-deletion-count-bg: var(--clr-diff-deletion-contrast-2-count-bg);
		--clr-diff-deletion-count-text: var(--clr-diff-deletion-contrast-2-count-text);
		--clr-diff-deletion-count-border: var(--clr-diff-deletion-contrast-2-count-border);
		/* addition */
		--clr-diff-addition-line-bg: var(--clr-diff-addition-contrast-2-line-bg);
		--clr-diff-addition-line-highlight: var(--clr-diff-addition-contrast-2-line-highlight);
		--clr-diff-addition-count-bg: var(--clr-diff-addition-contrast-2-count-bg);
		--clr-diff-addition-count-text: var(--clr-diff-addition-contrast-2-count-text);
		--clr-diff-addition-count-border: var(--clr-diff-addition-contrast-2-count-border);
	}

	.contrast-strong {
		--clr-diff-count-text: var(--clr-diff-count-text-contrast-3);
		/* deletion */
		--clr-diff-deletion-line-bg: var(--clr-diff-deletion-contrast-3-line-bg);
		--clr-diff-deletion-line-highlight: var(--clr-diff-deletion-contrast-3-line-highlight);
		--clr-diff-deletion-count-bg: var(--clr-diff-deletion-contrast-3-count-bg);
		--clr-diff-deletion-count-text: var(--clr-diff-deletion-contrast-3-count-text);
		--clr-diff-deletion-count-border: var(--clr-diff-deletion-contrast-3-count-border);
		/* addition */
		--clr-diff-addition-line-bg: var(--clr-diff-addition-contrast-3-line-bg);
		--clr-diff-addition-line-highlight: var(--clr-diff-addition-contrast-3-line-highlight);
		--clr-diff-addition-count-bg: var(--clr-diff-addition-contrast-3-count-bg);
		--clr-diff-addition-count-text: var(--clr-diff-addition-contrast-3-count-text);
		--clr-diff-addition-count-border: var(--clr-diff-addition-contrast-3-count-border);
	}
</style>
