<script lang="ts">
	import { listen } from '$lib/backend/ipc';
	import { Project } from '$lib/backend/projects';
	import { BaseBranch, NoDefaultTarget } from '$lib/baseBranch/baseBranch';
	import { BaseBranchService } from '$lib/baseBranch/baseBranchService';
	import { BranchDragActionsFactory } from '$lib/branches/dragActions';
	import { BranchService, createBranchServiceStore } from '$lib/branches/service';
	import { CommitDragActionsFactory } from '$lib/commits/dragActions';
	import NoBaseBranch from '$lib/components/NoBaseBranch.svelte';
	import NotOnGitButlerBranch from '$lib/components/NotOnGitButlerBranch.svelte';
	import ProblemLoadingRepo from '$lib/components/ProblemLoadingRepo.svelte';
	import ProjectSettingsMenuAction from '$lib/components/ProjectSettingsMenuAction.svelte';
	import { ReorderDropzoneManagerFactory } from '$lib/dragging/reorderDropzoneManager';
	import { DefaultGitHostFactory } from '$lib/gitHost/gitHostFactory';
	import { octokitFromAccessToken } from '$lib/gitHost/github/octokit';
	import { createGitHostStore } from '$lib/gitHost/interface/gitHost';
	import { createGitHostListingServiceStore } from '$lib/gitHost/interface/gitHostListingService';
	import History from '$lib/history/History.svelte';
	import { HistoryService } from '$lib/history/history';
	import MetricsReporter from '$lib/metrics/MetricsReporter.svelte';
	import Navigation from '$lib/navigation/Navigation.svelte';
	import { persisted } from '$lib/persisted/persisted';
	import { RemoteBranchService } from '$lib/stores/remoteBranches';
	import { parseRemoteUrl } from '$lib/url/gitUrl';
	import { debounce } from '$lib/utils/debounce';
	import * as events from '$lib/utils/events';
	import { createKeybind } from '$lib/utils/hotkeys';
	import { unsubscribe } from '$lib/utils/unsubscribe';
	import { BranchController } from '$lib/vbranches/branchController';
	import { VirtualBranchService } from '$lib/vbranches/virtualBranch';
	import { onDestroy, onMount, setContext, type Snippet } from 'svelte';
	import type { LayoutData } from './$types';

	const { data, children }: { data: LayoutData; children: Snippet } = $props();

	const {
		vbranchService,
		project,
		projectId,
		projectService,
		projectMetrics,
		baseBranchService,
		remoteBranchService,
		headService,
		userService,
		fetchSignal
	} = $derived(data);

	const branchesError = $derived(vbranchService.branchesError);
	const baseBranch = $derived(baseBranchService.base);
	const remoteUrl = $derived($baseBranch?.remoteUrl);
	const user = $derived(userService.user);
	const accessToken = $derived($user?.github_access_token);
	const baseError = $derived(baseBranchService.error);
	const projectError = $derived(projectService.error);

	$effect.pre(() => {
		setContext(HistoryService, data.historyService);
		setContext(VirtualBranchService, data.vbranchService);
		setContext(BranchController, data.branchController);
		setContext(BaseBranchService, data.baseBranchService);
		setContext(BaseBranch, baseBranch);
		setContext(Project, project);
		setContext(BranchDragActionsFactory, data.branchDragActionsFactory);
		setContext(CommitDragActionsFactory, data.commitDragActionsFactory);
		setContext(ReorderDropzoneManagerFactory, data.reorderDropzoneManagerFactory);
		setContext(RemoteBranchService, data.remoteBranchService);
	});

	let intervalId: any;

	const showHistoryView = persisted(false, 'showHistoryView');

	const octokit = $derived(accessToken ? octokitFromAccessToken(accessToken) : undefined);
	const gitHostFactory = $derived(octokit ? new DefaultGitHostFactory(octokit) : undefined);
	const repoInfo = $derived(remoteUrl ? parseRemoteUrl(remoteUrl) : undefined);

	const listServiceStore = createGitHostListingServiceStore(undefined);
	const githubRepoServiceStore = createGitHostStore(undefined);
	const branchServiceStore = createBranchServiceStore(undefined);

	// Refresh base branch if git fetch event is detected.
	const head = $derived(headService.head);
	const gbBranchActive = $derived($head === 'gitbutler/integration');

	// We end up with a `state_unsafe_mutation` when switching projects if we
	// don't use $effect.pre here.
	// TODO: can we eliminate the need to debounce?
	const fetch = $derived(fetchSignal.event);
	const debouncedBaseBranchResfresh = debounce(() => baseBranchService.refresh(), 500);
	$effect.pre(() => {
		if ($fetch || $head) debouncedBaseBranchResfresh();
	});

	// TODO: can we eliminate the need to debounce?
	const debouncedRemoteBranchRefresh = debounce(() => remoteBranchService.refresh(), 500);
	$effect.pre(() => {
		if ($baseBranch || $head || $fetch) debouncedRemoteBranchRefresh();
	});

	$effect.pre(() => {
		const gitHost = repoInfo ? gitHostFactory?.build(repoInfo) : undefined;
		const ghListService = gitHost?.listService();

		listServiceStore.set(ghListService);
		githubRepoServiceStore.set(gitHost);
		branchServiceStore.set(new BranchService(vbranchService, remoteBranchService, ghListService));
	});

	// Once on load and every time the project id changes
	$effect(() => {
		if (projectId) setupFetchInterval();
	});

	function setupFetchInterval() {
		baseBranchService.fetchFromRemotes();
		clearFetchInterval();
		const intervalMs = 15 * 60 * 1000; // 15 minutes
		intervalId = setInterval(async () => {
			await baseBranchService.fetchFromRemotes();
		}, intervalMs);
	}

	function clearFetchInterval() {
		if (intervalId) clearInterval(intervalId);
	}

	onMount(() => {
		const unsubscribe = listen<string>('menu://project/history/clicked', () => {
			$showHistoryView = !$showHistoryView;
		});

		return async () => {
			unsubscribe();
		};
	});

	const handleKeyDown = createKeybind({
		'$mod+Shift+H': () => {
			$showHistoryView = !$showHistoryView;
		}
	});

	onMount(() => {
		return unsubscribe(
			events.on('openHistory', () => {
				$showHistoryView = true;
			})
		);
	});

	onDestroy(() => clearFetchInterval());
</script>

<svelte:window on:keydown={handleKeyDown} />

<!-- forces components to be recreated when projectId changes -->
{#key projectId}
	<ProjectSettingsMenuAction />

	{#if !project}
		<p>Project not found!</p>
	{:else if $baseError instanceof NoDefaultTarget}
		<!-- Note that this requires the redirect above to work -->
		<NoBaseBranch />
	{:else if $baseError}
		<ProblemLoadingRepo error={$baseError} />
	{:else if $branchesError}
		<ProblemLoadingRepo error={$branchesError} />
	{:else if $projectError}
		<ProblemLoadingRepo error={$projectError} />
	{:else if !gbBranchActive && $baseBranch}
		<NotOnGitButlerBranch baseBranch={$baseBranch} />
	{:else if $baseBranch}
		<div class="view-wrap" role="group" ondragover={(e) => e.preventDefault()}>
			<Navigation />
			{#if $showHistoryView}
				<History on:hide={() => ($showHistoryView = false)} />
			{/if}
			{@render children()}
		</div>
	{/if}
	<MetricsReporter {projectMetrics} />
{/key}

<style>
	.view-wrap {
		position: relative;
		display: flex;
		width: 100%;
	}
</style>
