<script lang="ts">
	import PassphraseBox from './PassphraseBox.svelte';
	import PushButton, { BranchAction } from '../components/PushButton.svelte';
	import emptyStateImg from '$lib/assets/empty-state/commits-up-to-date.svg?raw';
	import { PromptService } from '$lib/backend/prompt';
	import { getGitHostChecksMonitor } from '$lib/gitHost/interface/gitHostChecksMonitor';
	import { getGitHostListingService } from '$lib/gitHost/interface/gitHostListingService';
	import { getGitHostPrMonitor } from '$lib/gitHost/interface/gitHostPrMonitor';
	import { project } from '$lib/testing/fixtures';
	import { getContext, getContextStore } from '$lib/utils/context';
	import { intersectionObserver } from '$lib/utils/intersectionObserver';
	import { BranchController } from '$lib/vbranches/branchController';
	import {
		getLocalCommits,
		getLocalAndRemoteCommits,
		getRemoteCommits
	} from '$lib/vbranches/contexts';
	import { VirtualBranch } from '$lib/vbranches/types';

	const branchController = getContext(BranchController);
	const promptService = getContext(PromptService);
	const branch = getContextStore(VirtualBranch);

	const listingService = getGitHostListingService();
	const prMonitor = getGitHostPrMonitor();
	const checksMonitor = getGitHostChecksMonitor();

	const [prompt, promptError] = promptService.reactToPrompt({
		branchId: $branch.id,
		timeoutMs: 30000
	});

	const localCommits = getLocalCommits();
	const localAndRemoteCommits = getLocalAndRemoteCommits();
	const remoteCommits = getRemoteCommits();

	let isLoading: boolean;
	let isInViewport = false;

	$: canBePushed = $localCommits.length !== 0 || $remoteCommits.length !== 0;
	$: hasRemoteCommits = $remoteCommits.length > 0;
	$: hasCommits =
		$localCommits.length > 0 || $localAndRemoteCommits.length > 0 || $remoteCommits.length > 0;
</script>

{#if hasCommits}
	<div
		class="actions"
		class:sticky={canBePushed}
		class:not-in-viewport={!isInViewport}
		use:intersectionObserver={{
			callback: (entry) => {
				if (entry.isIntersecting) {
					isInViewport = true;
				} else {
					isInViewport = false;
				}
			},
			options: {
				root: null,
				rootMargin: '-1px',
				threshold: 0
			}
		}}
	>
		{#if canBePushed}
			{#if $prompt}
				<PassphraseBox prompt={$prompt} error={$promptError} />
			{/if}
			<PushButton
				wide
				projectId={project.id}
				requiresForce={$branch.requiresForce}
				integrate={hasRemoteCommits}
				{isLoading}
				on:trigger={async (e) => {
					isLoading = true;
					try {
						if (e.detail.action === BranchAction.Push) {
							await branchController.pushBranch($branch.id, $branch.requiresForce);
							$listingService?.refresh();
							$prMonitor?.refresh();
							$checksMonitor?.update();
						} else if (e.detail.action === BranchAction.Integrate) {
							await branchController.mergeUpstream($branch.id);
						}
					} catch (e) {
						console.error(e);
					} finally {
						isLoading = false;
					}
				}}
			/>
		{:else}
			<div class="empty-state">
				<span class="text-base-body-12 empty-state__text"
					>Your branch is up to date with the remote.</span
				>

				<i class="empty-state__image">
					{@html emptyStateImg}
				</i>
			</div>
		{/if}
	</div>
{/if}

<style lang="postcss">
	.actions {
		background: var(--clr-bg-1);
		padding: 16px;
		border-top: 1px solid var(--clr-border-2);
		border-radius: 0 0 var(--radius-m) var(--radius-m);
	}

	/* EMPTY STATE */

	.empty-state {
		display: flex;
		align-items: center;
		gap: 20px;
	}

	.empty-state__image {
		flex-shrink: 0;
	}

	.empty-state__text {
		color: var(--clr-text-3);
		flex: 1;
	}

	/* MODIFIERS */
	.sticky {
		z-index: var(--z-lifted);
		position: sticky;
		bottom: 0;
	}

	.not-in-viewport {
		border-radius: 0;
	}
</style>
