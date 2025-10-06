<script lang="ts">
  import { format } from 'date-fns'
  import { Button as ButtonPrimitive } from 'bits-ui'
  import { goto } from '$app/navigation'
  import { cn } from '$lib/utils'
  import { onMount } from 'svelte'

  export let prevDate: Date
  export let currentDate: Date
  export let nextDate: Date

  export let prevEpisodes: any[] = []
  export let currentEpisodes: any[] = []
  export let nextEpisodes: any[] = []

  export let onSelectDate: (date: Date) => void
  export let onSelectEpisode: (ep: any) => void
  export let androidPortrait: boolean = false
  export let androidLandscape: boolean = false

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

  // Android portrait: collapse current-day list to 6 items with More+ button
  let showAllCurrent = false
  $: visibleCurrentEpisodes = androidPortrait && !showAllCurrent ? currentEpisodes.slice(0, 6) : currentEpisodes

  // Apply same behavior to previous and next day in Android portrait
  let showAllPrev = false
  let showAllNext = false
  $: visiblePrevEpisodes = androidPortrait && !showAllPrev ? prevEpisodes.slice(0, 6) : prevEpisodes
  $: visibleNextEpisodes = androidPortrait && !showAllNext ? nextEpisodes.slice(0, 6) : nextEpisodes

  // Below 1400px: change 3-day list to 2-day (current + next). Hide Previous.
  let wide1400 = false
  onMount(() => {
    const mq = window.matchMedia('(min-width: 1400px)')
    const update = () => { wide1400 = mq.matches }
    update()
    mq.addEventListener('change', update)
    return () => mq.removeEventListener('change', update)
  })

  // Reset expanded state when dates change
  let _prevKey = 0, _currentKey = 0, _nextKey = 0
  $: { const k = +prevDate; if (k !== _prevKey) { _prevKey = k; showAllPrev = false } }
  $: { const k = +currentDate; if (k !== _currentKey) { _currentKey = k; showAllCurrent = false } }
  $: { const k = +nextDate; if (k !== _nextKey) { _nextKey = k; showAllNext = false } }
</script>

<div class={cn(
  androidPortrait
    ? 'flex flex-col gap-3 h-full mt-2'
    : (wide1400 ? (androidLandscape ? 'grid grid-cols-3 gap-2 h-full mt-2' : 'grid grid-cols-3 gap-3 h-full mt-2')
                : (androidLandscape ? 'grid grid-cols-2 gap-2 h-full mt-2' : 'grid grid-cols-2 gap-3 h-full mt-2'))
)}>
  <!-- Previous Day -->
  {#if wide1400}
  <div class={cn('flex flex-col border border-white/15 bg-muted/20 rounded-lg overflow-hidden hover:bg-accent/50', androidPortrait ? 'min-h-0 mt-0' : 'min-h-150 min-w-[215px] mt-2')}>
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
        {#each visiblePrevEpisodes as ep (ep.id + ':' + ep.episode)}
          <ButtonPrimitive.Root href={'/app/anime/' + ep.id} class={cn('flex items-center w-full text-neutral-200 group px-1 py-1 rounded-md')}
            on:click={(e) => handleEpisodeClick(e, ep)}>
            <div class={cn('w-12 pl-0.4 tabular-nums text-neutral-400 group-hover:text-neutral-100', androidPortrait ? 'text-[11px]' : androidLandscape ? 'text-[12px]' : 'text-xs')}>{format(ep.airTime, 'HH:mm')}</div>
            <div class={cn('flex-1 pl-1.5 font-medium overflow-hidden', androidLandscape ? 'text-xs' : 'text-sm', +ep.airTime < Date.now() && 'line-through')} style="display:-webkit-box;-webkit-line-clamp:2;-webkit-box-orient:vertical;hyphens:auto;word-break:break-word;">
              {ep.title?.userPreferred}
            </div>
            <div class={cn('ml-auto pr-0.3 text-nowrap shrink-0', androidPortrait ? 'text-sm' : androidLandscape ? 'text-sm' : 'text-md')}>Ep {ep.episode}</div>
          </ButtonPrimitive.Root>
        {/each}
        {#if androidPortrait && prevEpisodes.length > 6 && !showAllPrev}
          <div class="flex justify-center py-1">
            <button type="button" class="px-3 py-1 rounded-md border text-sm opacity-90 hover:opacity-100"
              on:click={(e) => { e.preventDefault(); e.stopPropagation(); showAllPrev = true }}>More+</button>
          </div>
        {/if}
      {/if}
    </div>
  </div>
  {/if}
  <!-- Current Day (Selected) -->
  <div class={cn('flex flex-col border-2 border-white/25 bg-muted/40 rounded-lg overflow-hidden scale-[1.0005]', androidPortrait ? 'min-h-0' : 'min-h-0 min-w-[220px]')}>
    <button type="button" class="px-3 py-2.5 text-base font-semibold text-neutral-100 bg-muted/90 cursor-pointer select-none text-center w-full" on:click={selectCurrentDate} aria-label="Current day">
      {format(currentDate, 'EEE d')}
    </button>
    <div class="flex-1 overflow-y-auto pt-2 px-1 pb-1 space-y-1.5" style="content-visibility:auto; contain-intrinsic-size: 320px 400px;">
      {#if currentEpisodes.length === 0}
        <div class="text-sm text-muted-foreground px-2 py-8 text-center">No episodes</div>
      {:else}
        {#each visibleCurrentEpisodes as ep (ep.id + ':' + ep.episode)}
          <ButtonPrimitive.Root href={'/app/anime/' + ep.id} class={cn('flex items-center w-full group px-1 py-1 rounded-md')}
            on:click={(e) => handleEpisodeClick(e, ep)}>
            <div class={cn('w-12 pl-0.4 tabular-nums text-neutral-400 group-hover:text-neutral-200', androidPortrait ? 'text-[11px]' : androidLandscape ? 'text-[12px]' : 'text-xs md:text-sm')}>{format(ep.airTime, 'HH:mm')}</div>
            <div class={cn('flex-1 pl-1.5 font-medium overflow-hidden', androidLandscape ? 'text-xs' : 'text-sm', +ep.airTime < Date.now() && 'line-through')} style="display:-webkit-box;-webkit-line-clamp:2;-webkit-box-orient:vertical;hyphens:auto;word-break:break-word;">
              {ep.title?.userPreferred}
            </div>
            <div class={cn('ml-auto pr-0.3 text-nowrap shrink-0', androidPortrait ? 'text-sm' : androidLandscape ? 'text-sm' : 'md:text-base')}>Ep {ep.episode}</div>
          </ButtonPrimitive.Root>
        {/each}
        {#if androidPortrait && currentEpisodes.length > 6 && !showAllCurrent}
          <div class="flex justify-center py-1">
            <button type="button" class="px-3 py-1 rounded-md border text-sm opacity-90 hover:opacity-100" on:click={() => showAllCurrent = true}>More+</button>
          </div>
        {/if}
      {/if}
    </div>
  </div>

  <!-- Next Day -->
  <div class={cn('flex flex-col border border-white/15 bg-muted/20 rounded-lg overflow-hidden hover:bg-accent/25', androidPortrait ? 'min-h-0 mt-0' : 'min-h-0 min-w-[215px] mt-2')}>
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
        {#each visibleNextEpisodes as ep (ep.id + ':' + ep.episode)}
          <ButtonPrimitive.Root href={'/app/anime/' + ep.id} class={cn('flex items-center text-neutral-200 w-full group px-1 py-1 rounded-md')}
            on:click={(e) => handleEpisodeClick(e, ep)}>
            <div class={cn('w-12 pl-0.4 tabular-nums text-neutral-400 group-hover:text-neutral-200', androidPortrait ? 'text-[11px]' : androidLandscape ? 'text-[12px]' : 'text-xs')}>{format(ep.airTime, 'HH:mm')}</div>
            <div class={cn('flex-1 pl-1.5 font-medium overflow-hidden', androidLandscape ? 'text-xs' : 'text-sm', +ep.airTime < Date.now() && 'line-through')} style="display:-webkit-box;-webkit-line-clamp:2;-webkit-box-orient:vertical;hyphens:auto;word-break:break-word;">
              {ep.title?.userPreferred}
            </div>
            <div class={cn('ml-auto pr-0.3 text-nowrap shrink-0', androidPortrait ? 'text-sm' : androidLandscape ? 'text-sm' : 'text-md')}>Ep {ep.episode}</div>
          </ButtonPrimitive.Root>
        {/each}
        {#if androidPortrait && nextEpisodes.length > 6 && !showAllNext}
          <div class="flex justify-center py-1">
            <button type="button" class="px-3 py-1 rounded-md border text-sm opacity-90 hover:opacity-100"
              on:click={(e) => { e.preventDefault(); e.stopPropagation(); showAllNext = true }}>More+</button>
          </div>
        {/if}
      {/if}
    </div>
  </div>
</div>
