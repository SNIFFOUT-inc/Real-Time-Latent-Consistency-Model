<script lang="ts">
  import { lcmLiveStatus, LCMLiveStatus, streamId } from '$lib/lcmLive';
  import { getPipelineValues } from '$lib/store';
  import Expand from '$lib/icons/expand.svelte';
  import { snapImage, expandWindow } from '$lib/utils';

  $: isLCMRunning = $lcmLiveStatus !== LCMLiveStatus.DISCONNECTED;
  $: console.log('isLCMRunning', isLCMRunning);
  let imageEl: HTMLImageElement;
  let expandedWindow: Window;
  let isExpanded = false;
</script>

<div
  class="relative mx-auto aspect-square max-w-fill self-center overflow-hidden rounded-lg border border-slate-300"
>
  <!-- svelte-ignore a11y-missing-attribute -->
  {#if isLCMRunning}
    {#if !isExpanded}
      <img
        bind:this={imageEl}
        class="aspect-square w-full rounded-lg"
        src={'/api/stream/' + $streamId}
      />
    {/if}
  {:else}
    <img
      class="aspect-square w-full rounded-lg"
      src="data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAQAAAC1HAwCAAAAC0lEQVR42mNkYAAAAAYAAjCB0C8AAAAASUVORK5CYII="
    />
  {/if}
</div>
