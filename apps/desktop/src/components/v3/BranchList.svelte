<script lang="ts">
	import CardOverlay from '$components/CardOverlay.svelte';
	import Dropzone from '$components/Dropzone.svelte';
	import ReduxResult from '$components/ReduxResult.svelte';
	import SeriesHeaderContextMenu from '$components/SeriesHeaderContextMenu.svelte';
	import StackStickyButtons from '$components/StackStickyButtons.svelte';
	import BranchCard from '$components/v3/BranchCard.svelte';
	import BranchCommitList from '$components/v3/BranchCommitList.svelte';
	import CommitGoesHere from '$components/v3/CommitGoesHere.svelte';
	import CommitRow from '$components/v3/CommitRow.svelte';
	import NewBranchModal from '$components/v3/NewBranchModal.svelte';
	import PublishButton from '$components/v3/PublishButton.svelte';
	import PushButton from '$components/v3/PushButton.svelte';
	import {
		getColorFromCommitState,
		getIconFromCommitState,
		isLocalAndRemoteCommit,
		isUpstreamCommit
	} from '$components/v3/lib';
	import BaseBranchService from '$lib/baseBranch/baseBranchService.svelte';
	import {
		AmendCommitWithChangeDzHandler,
		SquashCommitDzHandler,
		type DzCommitData
	} from '$lib/commits/dropHandler';
	import { DefaultForgeFactory } from '$lib/forge/forgeFactory.svelte';
	import { StackService } from '$lib/stacks/stackService.svelte';
	import { combineResults } from '$lib/state/helpers';
	import { UiState } from '$lib/state/uiState.svelte';
	import { openExternalUrl } from '$lib/utils/url';
	import { inject } from '@gitbutler/shared/context';

	type Props = {
		projectId: string;
		stackId: string;
	};

	const { projectId, stackId }: Props = $props();
	const [stackService, uiState, baseBranchService, forge] = inject(
		StackService,
		UiState,
		BaseBranchService,
		DefaultForgeFactory
	);

	const branchesResult = $derived(stackService.branches(projectId, stackId));

	const baseBranchResponse = $derived(baseBranchService.baseBranch(projectId));
	const base = $derived(baseBranchResponse.current.data);
	const baseSha = $derived(base?.baseSha);

	const drawer = $derived(uiState.project(projectId).drawerPage);
	const isCommitting = $derived(drawer.current === 'new-commit');

	const selection = $derived(uiState.stack(stackId).selection.get());
	const selectedCommitId = $derived(selection.current?.commitId);
	const selectedBranchName = $derived(selection.current?.branchName);

	let newBranchModal = $state<ReturnType<typeof NewBranchModal>>();
</script>

<ReduxResult {stackId} {projectId} result={branchesResult.current}>
	{#snippet children(branches, { stackId, projectId })}
		{#each branches as branch, i}
			{@const branchName = branch.name}
			{@const localAndRemoteCommits = stackService.commits(projectId, stackId, branchName)}
			{@const upstreamOnlyCommits = stackService.upstreamCommits(projectId, stackId, branchName)}
			{@const branchDetailsResult = stackService.branchDetails(projectId, stackId, branchName)}
			{@const commitResult = stackService.commitAt(projectId, stackId, branchName, 0)}
			{@const first = i === 0}
			{@const last = i === branches.length - 1}

			<ReduxResult
				{projectId}
				{stackId}
				result={combineResults(
					localAndRemoteCommits.current,
					upstreamOnlyCommits.current,
					branchDetailsResult.current,
					commitResult.current
				)}
			>
				{#snippet children([localAndRemoteCommits, upstreamOnlyCommits, branchDetails, commit])}
					{@const branchType = commit?.state.type || 'LocalOnly'}
					{@const iconName = getIconFromCommitState(commit?.id, commit?.state)}
					{@const lineColor = commit
						? getColorFromCommitState(commit.id, commit.state)
						: 'var(--clr-commit-local)'}
					{@const isNewBranch =
						upstreamOnlyCommits.length === 0 && localAndRemoteCommits.length === 0}
					<BranchCard
						{projectId}
						{stackId}
						{branchName}
						{lineColor}
						{iconName}
						{first}
						{last}
						{isNewBranch}
						{isCommitting}
						pushStatus={branchDetails.pushStatus}
						isConflicted={branchDetails.isConflicted}
						lastUpdatedAt={branchDetails.lastUpdatedAt}
						reviewId={branch.reviewId || undefined}
						prNumber={branch.prNumber || undefined}
						trackingBranch={branch.remoteTrackingBranch || undefined}
						headCommit={branchDetails.commits[0]?.id || branchDetails.baseCommit}
					>
						{#snippet commitList()}
							<BranchCommitList {projectId} {stackId} {branchName} {selectedCommitId}>
								{#snippet empty()}
									{#if isCommitting}
										<CommitGoesHere
											selected={branchName === selectedBranchName}
											first
											last
											onclick={() =>
												uiState.stack(stackId).selection.set({
													branchName,
													commitId: branchDetails.baseCommit
												})}
										/>
									{/if}
								{/snippet}
								{#snippet upstreamTemplate({ commit, first, lastCommit, selected })}
									{@const commitId = commit.id}
									{#if !isCommitting}
										<CommitRow
											{stackId}
											{branchName}
											{projectId}
											{first}
											lastCommit={lastCommit && localAndRemoteCommits.length === 0}
											{commit}
											{selected}
											onclick={() => {
												uiState
													.stack(stackId)
													.selection.set({ branchName, commitId, upstream: true });
												uiState.project(projectId).drawerPage.set(undefined);
											}}
										/>
									{/if}
								{/snippet}
								{#snippet localAndRemoteTemplate({
									commit,
									first,
									last,
									lastCommit,
									selectedCommitId
								})}
									{@const commitId = commit.id}
									{@const selected =
										commit.id === selectedCommitId && branchName === selectedBranchName}
									{#if isCommitting}
										<!-- Only commits to the base can be `last`, see next `CommitGoesHere`. -->
										<CommitGoesHere
											{selected}
											{first}
											last={false}
											onclick={() => uiState.stack(stackId).selection.set({ branchName, commitId })}
										/>
									{/if}
									{@const dzCommit: DzCommitData = {
										id: commit.id,
										isRemote: isUpstreamCommit(commit),
										isIntegrated: isLocalAndRemoteCommit(commit) && commit.state.type === 'Integrated',
										isConflicted: isLocalAndRemoteCommit(commit) && commit.hasConflicts,
									}}
									{@const amendHandler = new AmendCommitWithChangeDzHandler(
										projectId,
										stackService,
										stackId,
										dzCommit,
										(newId) => uiState.stack(stackId).selection.set({ branchName, commitId: newId })
									)}
									{@const squashHandler = new SquashCommitDzHandler({
										stackService,
										projectId,
										stackId,
										commit: dzCommit
									})}
									<Dropzone handlers={[amendHandler, squashHandler]}>
										{#snippet overlay({ hovered, activated, handler })}
											{@const label =
												handler instanceof AmendCommitWithChangeDzHandler ? 'Amend' : 'Squash'}
											<CardOverlay {hovered} {activated} {label} />
										{/snippet}
										<CommitRow
											{stackId}
											{branchName}
											{projectId}
											{first}
											{lastCommit}
											lastBranch={last}
											{commit}
											{selected}
											draggable
											onclick={() => {
												const stackState = uiState.stack(stackId);
												stackState.selection.set({ branchName, commitId });
												stackState.activeSelectionId.set({ type: 'commit', commitId });
												uiState.project(projectId).drawerPage.set(undefined);
											}}
										/>
									</Dropzone>
									{#if isCommitting && last}
										<CommitGoesHere
											{first}
											{last}
											selected={selectedCommitId === baseSha && branchName === selectedBranchName}
											onclick={() =>
												uiState.stack(stackId).selection.set({ branchName, commitId: baseSha })}
										/>
									{/if}
								{/snippet}
							</BranchCommitList>
						{/snippet}
						{#snippet menu()}
							{@const forgeBranch = branch.remoteTrackingBranch
								? forge.current?.branch(branch.remoteTrackingBranch)
								: undefined}
							<SeriesHeaderContextMenu
								{projectId}
								{stackId}
								{branchType}
								branchName={branch.name}
								seriesCount={branches.length}
								isTopBranch={first}
								descriptionOption={false}
								onGenerateBranchName={() => {
									throw new Error('Not implemented!');
								}}
								onAddDependentSeries={() => newBranchModal?.show(branchName)}
								onOpenInBrowser={() => {
									const url = forgeBranch?.url;
									if (url) openExternalUrl(url);
								}}
								isPushed={!!branch.remoteTrackingBranch}
							/>
						{/snippet}
					</BranchCard>
				{/snippet}
			</ReduxResult>
		{/each}
		<StackStickyButtons>
			<PushButton {projectId} {stackId} multipleBranches={branches.length > 1} />
			<PublishButton {projectId} {stackId} {branches} />
		</StackStickyButtons>
	{/snippet}
</ReduxResult>

<NewBranchModal {projectId} {stackId} bind:this={newBranchModal} />
