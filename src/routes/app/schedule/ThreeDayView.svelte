<script lang="ts">
  import { format } from 'date-fns'
  import { Button as ButtonPrimitive } from 'bits-ui'
  import { goto } from '$app/navigation'
  import { cn } from '$lib/utils'

  export let prevDate: Date
  export let currentDate: Date
  export let nextDate: Date

  export let prevEpisodes: any[] = []
  export let currentEpisodes: any[] = []
  export let nextEpisodes: any[] = []

  export let onSelectDate: (date: Date) => void
  export let onSelectEpisode: (ep: any) => void

  let lastClickId: string | null = null
  let lastClickAt = 0

  function handleEpisodeClick(e: MouseEvent, ep: any) {
    e.preventDefault()
    e.stopPropagation()
    const id = `${ep.id}:${ep.episode}`
    const now = Date.now()
    if (lastClickId === id && (now - lastClickAt) <= 3000) {
      goto(`/app/anime/${ep.id}`)
    } else {
      lastClickId = id
      lastClickAt = now
      onSelectEpisode(ep)
    }
  }

  // Named handlers (avoids recreating lambdas in template)
  function selectPrevDate() { onSelectDate(prevDate) }
  function selectCurrentDate() { onSelectDate(currentDate) }
  function selectNextDate() { onSelectDate(nextDate) }

  function keydownSelectPrev(e: KeyboardEvent) { if (e.key === 'Enter' || e.key === ' ') { e.preventDefault(); e.stopPropagation(); onSelectDate(prevDate) } }
  function keydownSelectCurrent(e: KeyboardEvent) { if (e.key === 'Enter' || e.key === ' ') { e.preventDefault(); e.stopPropagation(); onSelectDate(currentDate) } }
  function keydownSelectNext(e: KeyboardEvent) { if (e.key === 'Enter' || e.key === ' ') { e.preventDefault(); e.stopPropagation(); onSelectDate(nextDate) } }
</script>

<div class="grid grid-cols-3 gap-3 h-full mt-2">
  <!-- Previous Day -->
  <div class="flex flex-col border border-white/15 bg-muted/20 rounded-lg overflow-hidden min-h-150 min-w-[215px] mt-2 hover:bg-accent/50">
    <button type="button" class="px-3 py-3 text-neutral-200 text-base font-semibold bg-muted/60 cursor-pointer select-none text-center w-full" on:click={selectPrevDate} aria-label="Previous day">
      {format(prevDate, 'EEE d')}
    </button>

    <div
      class="flex-1 overflow-y-auto pt-2 px-1 pb-1 space-y-1.5 cursor-pointer"
      role="button"
      tabindex="0"
      style="content-visibility:auto; contain-intrinsic-size: 320px 400px;"
      on:click={selectPrevDate}
      on:keydown={keydownSelectPrev}
    >
      {#if prevEpisodes.length === 0}
        <div class="text-sm text-muted-foreground px-2 py-8 text-center">No episodes</div>
      {:else}
        {#each prevEpisodes as ep (ep.id + ':' + ep.episode)}
          <ButtonPrimitive.Root href={'/app/anime/' + ep.id} class={cn('flex items-center w-full text-neutral-200 group px-1 py-1 rounded-md')}
            on:click={(e) => handleEpisodeClick(e, ep)}>
            <div class="w-12 pl-0.4 tabular-nums text-xs text-neutral-400 group-hover:text-neutral-100">{format(ep.airTime, 'HH:mm')}</div>
            <div class={cn('flex-1 pl-1.5 font-medium text-sm overflow-hidden', +ep.airTime < Date.now() && 'line-through')} style="display:-webkit-box;-webkit-line-clamp:2;-webkit-box-orient:vertical;hyphens:auto;word-break:break-word;">
              {ep.title?.userPreferred}
            </div>
            <div class="ml-auto pr-0.3 text-md text-nowrap shrink-0">Ep {ep.episode}</div>
          </ButtonPrimitive.Root>
        {/each}
      {/if}
    </div>
  </div>

  <!-- Current Day (Selected) -->
  <div class="flex flex-col border-2 border-white/25 bg-muted/40 rounded-lg overflow-hidden min-h-0 min-w-[220px] scale-[1.0005]">
    <button type="button" class="px-3 py-2.5 text-base font-semibold text-neutral-100 bg-muted/90 cursor-pointer select-none text-center w-full" on:click={selectCurrentDate} aria-label="Current day">
      {format(currentDate, 'EEE d')}
    </button>
    <div class="flex-1 overflow-y-auto pt-2 px-1 pb-1 space-y-1.5" style="content-visibility:auto; contain-intrinsic-size: 320px 400px;">
      {#if currentEpisodes.length === 0}
        <div class="text-sm text-muted-foreground px-2 py-8 text-center">No episodes</div>
      {:else}
        {#each currentEpisodes as ep (ep.id + ':' + ep.episode)}
          <ButtonPrimitive.Root href={'/app/anime/' + ep.id} class={cn('flex items-center w-full group px-1 py-1 rounded-md')}
            on:click={(e) => handleEpisodeClick(e, ep)}>
            <div class="w-12 pl-0.4 tabular-nums text-xs md:text-sm text-neutral-400 group-hover:text-neutral-200">{format(ep.airTime, 'HH:mm')}</div>
            <div class={cn('flex-1 pl-1.5 font-medium text-sm overflow-hidden', +ep.airTime < Date.now() && 'line-through')} style="display:-webkit-box;-webkit-line-clamp:2;-webkit-box-orient:vertical;hyphens:auto;word-break:break-word;">
              {ep.title?.userPreferred}
            </div>
            <div class="ml-auto pr-0.3 md:text-base text-nowrap shrink-0">Ep {ep.episode}</div>
          </ButtonPrimitive.Root>
        {/each}
      {/if}
    </div>
  </div>

  <!-- Next Day -->
  <div class="flex flex-col border border-white/15 bg-muted/20 rounded-lg overflow-hidden min-h-0 min-w-[215px] mt-2 hover:bg-accent/25">
    <button type="button" class="px-3 py-3 text-base text-neutral-200 font-semibold bg-muted/60 cursor-pointer select-none text-center w-full" on:click={selectNextDate} aria-label="Next day">
      {format(nextDate, 'EEE d')}
    </button>
    <div
      class="flex-1 overflow-y-auto pt-2 px-1 pb-1 space-y-1.5 cursor-pointer"
      role="button"
      tabindex="0"
      style="content-visibility:auto; contain-intrinsic-size: 320px 400px;"
      on:click={selectNextDate}
      on:keydown={keydownSelectNext}
    >
      {#if nextEpisodes.length === 0}
        <div class="text-sm text-muted-foreground px-2 py-8 text-center">No episodes</div>
      {:else}
        {#each nextEpisodes as ep (ep.id + ':' + ep.episode)}
          <ButtonPrimitive.Root href={'/app/anime/' + ep.id} class={cn('flex items-center text-neutral-200 w-full group px-1 py-1 rounded-md')}
            on:click={(e) => handleEpisodeClick(e, ep)}>
            <div class="w-12 pl-0.4 tabular-nums text-xs text-neutral-400 group-hover:text-neutral-200">{format(ep.airTime, 'HH:mm')}</div>
            <div class={cn('flex-1 pl-1.5 font-medium overflow-hidden', +ep.airTime < Date.now() && 'line-through')} style="display:-webkit-box;-webkit-line-clamp:2;-webkit-box-orient:vertical;hyphens:auto;word-break:break-word;">
              {ep.title?.userPreferred}
            </div>
            <div class="ml-auto pr-0.3 text-md text-nowrap shrink-0">Ep {ep.episode}</div>
          </ButtonPrimitive.Root>
        {/each}
      {/if}
    </div>
  </div>
</div>
