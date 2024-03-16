<script lang="ts">
  import { onMount } from 'svelte';
  import type { Fields, PipelineInfo } from '$lib/types';
  import { PipelineMode } from '$lib/types';
  import ImagePlayer from '$lib/components/ImagePlayer.svelte';
  import VideoInput from '$lib/components/VideoInput.svelte';
  import Button from '$lib/components/Button.svelte';
  import PipelineOptions from '$lib/components/PipelineOptions.svelte';
  import Spinner from '$lib/icons/spinner.svelte';
  import Warning from '$lib/components/Warning.svelte';
  import { lcmLiveStatus, lcmLiveActions, LCMLiveStatus } from '$lib/lcmLive';
  import { mediaStreamActions, onFrameChangeStore } from '$lib/mediaStream';
  import { getPipelineValues, deboucedPipelineValues } from '$lib/store';
  import { fade } from 'svelte/transition';
  let pipelineParams: Fields;
  let pipelineInfo: PipelineInfo;
  let pageContent: string;
  let videoContent: string;
  let isImageMode: boolean = false;
  let maxQueueSize: number = 0;
  let currentQueueSize: number = 0;
  let queueCheckerRunning: boolean = false;
  let warningMessage: string = '';

  let time = new Date();
  function setTimer() {
    const interval = setInterval(() => {
			time = new Date();
      if (time.getSeconds() == 0) {
        toggleLcmLive();
        console.log(time);
      }
		}, 1000);

		return () => {
			clearInterval(interval);
		};
  }

  onMount(() => {
    getSettings();
    setTimer();
  });

  async function getSettings() {
    const settings = await fetch('/api/settings').then((r) => r.json());
    pipelineParams = settings.input_params.properties;
    pipelineInfo = settings.info.properties;
    isImageMode = pipelineInfo.input_mode.default === PipelineMode.IMAGE;
    maxQueueSize = settings.max_queue_size;
    pageContent = settings.page_content;
    videoContent = settings.video_content;
    console.log(pipelineParams);
    toggleQueueChecker(true);
  }
  function toggleQueueChecker(start: boolean) {
    queueCheckerRunning = start && maxQueueSize > 0;
    if (start) {
      getQueueSize();
    }
  }
  async function getQueueSize() {
    if (!queueCheckerRunning) {
      return;
    }
    const data = await fetch('/api/queue').then((r) => r.json());
    currentQueueSize = data.queue_size;
    setTimeout(getQueueSize, 10000);
  }

  function getSreamdata() {
    if (isImageMode) {
      return [getPipelineValues(), $onFrameChangeStore?.blob];
    } else {
      return [$deboucedPipelineValues];
    }
  }

  $: isLCMRunning = $lcmLiveStatus !== LCMLiveStatus.DISCONNECTED;
  $: if ($lcmLiveStatus === LCMLiveStatus.TIMEOUT) {
    warningMessage = 'Session timed out. Please try again.';
  }
  let disabled = false;
  async function toggleLcmLive() {
    try {
      if (!isLCMRunning) {
        if (isImageMode) {
          await mediaStreamActions.enumerateDevices();
          await mediaStreamActions.start();
        }
        disabled = true;
        await lcmLiveActions.start(getSreamdata);
        disabled = false;
        toggleQueueChecker(false);
        changePromptItem();
      } else {
        if (isImageMode) {
          mediaStreamActions.stop();
        }
        lcmLiveActions.stop();
        toggleQueueChecker(true);
        changeVideoItem();
      }
    } catch (e) {
      warningMessage = e instanceof Error ? e.message : '';
      disabled = false;
      toggleQueueChecker(true);
    }
  }

  async function changeVideoItem() {
    const data = await fetch('/api/change_video_item').then((r) => r.json());
    if (!data.result) {
      console.error("An error has occurred when changing Video Item");
    } else {
      videoContent = data.item
    }
  }

  async function changePromptItem() {
    const data = await fetch('/api/change_prompt_item').then((r) => r.json());
    if (!data.result) {
      console.error("An error has occurred when changing Prompt Item");
    }
  }
</script>

<svelte:head>
  <script
    src="https://cdnjs.cloudflare.com/ajax/libs/iframe-resizer/4.3.9/iframeResizer.contentWindow.min.js"
  ></script>
</svelte:head>

<main class="container mx-auto flex max-w-fit flex-col gap-3 px-4 py-4">
  <Warning bind:message={warningMessage}></Warning>
  <article>
    {#if pageContent}
      {@html pageContent}
    {/if}
    {#if pipelineParams}
      {#if isLCMRunning}
        <p class="image" in:fade={{duration: 1000}} out:fade={{duration: 1000}}>
          <ImagePlayer />
        </p>
        <p class="image" in:fade={{duration: 1000}} out:fade={{duration: 1000}}>
          {#if isImageMode}
            <VideoInput
              width={Number(pipelineParams.width.default)}
              height={Number(pipelineParams.height.default)}
            ></VideoInput>
          {/if}
        </p>
      {:else}
        <p class="video" in:fade={{duration: 1000}} out:fade={{duration: 1000}}>
          {#if videoContent}
            {@html videoContent}
          {/if}
        </p>
      {/if}
      <!-- will be deleted -->
      <div class="sm:col-span-4 sm:row-start-2">
        <Button on:click={toggleLcmLive} {disabled} classList={'text-lg my-1 p-2'}>
          {#if isLCMRunning}
            Stop
          {:else}
            Start
          {/if}
        </Button>
        <PipelineOptions {pipelineParams}></PipelineOptions>
      </div>
    {:else}
      <div class="flex items-center justify-center gap-3 py-48 text-2xl">
        <Spinner classList={'animate-spin opacity-50'}></Spinner>
        <p>Loading...</p>
      </div>
    {/if}
  </article>
</main>

<style lang="postcss">
  :global(html) {
    @apply text-black dark:bg-gray-900 dark:text-white;
  }
  p.image { padding: 0px 6.5% 1% 6.5%; }
  p.video { padding: 0px 1% 5% 1%; }
</style>
