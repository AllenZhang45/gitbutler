<script lang="ts">
	import Badge from '@gitbutler/ui/Badge.svelte';

	type Props = {
		first?: boolean;
		last?: boolean;
		draft?: boolean;
		selected: boolean;
		onclick?: () => void;
	};

	const { first, last, draft, selected, onclick }: Props = $props();
</script>

{#snippet indicator(args?: { last?: boolean; first?: boolean; draft?: boolean })}
	<div
		class="indicator"
		class:first={args?.first}
		class:last={args?.last}
		class:draft={args?.draft}
	>
		<div class="pin">
			<div class="pin__line"></div>
			<div class="pin__circle"></div>
		</div>
		<Badge size="tag" style="pop">Your commit goes here</Badge>
	</div>
{/snippet}
{#snippet commitHere(args: { last?: boolean })}
	{@const last = args?.last}
	<button class="commit-here" class:commit-here_last={last} type="button" {onclick}>
		<div class="commit-here__circle"></div>
		<div class="commit-here__line"></div>

		<div class="commit-here__label text-11 text-semibold">Commit here</div>
	</button>
{/snippet}

{#if selected}
	{@render indicator({ first, last, draft })}
{/if}
{#if !selected}
	{@render commitHere({ last })}
{/if}

<style lang="postcss">
	.indicator {
		padding: 12px 0;
		display: flex;
		gap: 12px;
		align-items: center;
		background-color: var(--clr-bg-1);
		border-bottom: 1px solid var(--clr-border-2);
		&.last {
			border-top: 1px solid var(--clr-border-2);
			border-bottom: none;
			border-radius: 0 0 var(--radius-l) var(--radius-l);
		}
		&.draft {
			border-top: none;
			border-bottom: none;
			border-radius: 0 0 var(--radius-l) var(--radius-l);
		}
	}
	.pin {
		display: flex;
		align-items: center;
		width: 40px;
		height: 10px;
		margin-left: -15px;
		position: relative;
	}
	.pin__line {
		flex-grow: 1;
		height: 2px;
		background-color: var(--clr-theme-pop-element);
	}
	.pin__circle {
		border-radius: 100%;
		width: 10px;
		height: 10px;
		outline: 2px solid var(--clr-theme-pop-element);
	}

	/* COMMIT HERE */
	.commit-here {
		width: 100%;
		position: relative;
		height: 20px;
		margin-top: -10px;
		margin-bottom: -10px;
		display: flex;
		align-items: center;
		opacity: 0;
		z-index: var(--z-lifted);
		transition: height var(--transition-fast);

		&:hover {
			opacity: 1;

			& .commit-here__label {
				opacity: 1;
				transform: translateY(-50%) translateX(0) scale(1);
			}

			&.commit-here_last {
				height: 30px;
				margin-bottom: 0;
			}
		}
	}
	.commit-here__circle {
		position: absolute;
		left: 16px;
		top: 50%;
		transform: translateY(-50%);
		border-radius: 100%;
		width: 10px;
		height: 10px;
		background-color: var(--clr-theme-pop-element);
		outline: 2px solid var(--clr-bg-1);
	}
	.commit-here__line {
		background-color: var(--clr-theme-pop-element);
		height: 2px;
		flex-grow: 1;
		margin-left: -15px;
	}
	.commit-here__label {
		position: absolute;
		top: 50%;
		left: 38px;
		padding: 3px 6px;
		border-radius: var(--radius-ml);
		background-color: var(--clr-theme-pop-element);
		color: var(--clr-core-ntrl-100);
		transform: translateY(-50%) translateX(-10%) scale(0.95);
		transition:
			opacity var(--transition-fast),
			transform var(--transition-medium);
	}

	/* .commit-here_last {
		margin-bottom: 0;
	} */
</style>
