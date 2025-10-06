<script lang="ts">
  import { format } from 'date-fns'
  import { Button as ButtonPrimitive } from 'bits-ui'
  import { onDestroy, onMount, tick } from 'svelte'
  import StatusDot from '$lib/components/StatusDot.svelte'
  import { list as listStatus } from '$lib/modules/auth'
  import { cn } from '$lib/utils'

  export let episodes: any[] = []
  export let selectedEpisode: any | undefined
  export let onSelectEpisode: (ep: any) => void
  export let highlightUntil: number = 0
  export let androidPortrait: boolean = false
  export let androidLandscape: boolean = false
  let localHighlightUntil = 0
  let prevHighlightUntil = 0
  let localTimer: any
  let shortenedOnce = false
  let listEl: HTMLDivElement | null = null

  // Layout breakpoint detection for 1600px
  let wide1600 = false
  onMount(() => {
    const mq = window.matchMedia('(min-width: 1600px)')
    const update = () => { wide1600 = mq.matches }
    update()
    mq.addEventListener('change', update)
    return () => mq.removeEventListener('change', update)
  })

  function coverSrcset(ci?: any) {
    const list: string[] = []
    if (ci?.medium) list.push(`${ci.medium} 320w`)
    if (ci?.large) list.push(`${ci.large} 640w`)
    if (ci?.extraLarge) list.push(`${ci.extraLarge} 1280w`)
    return list.join(', ')
  }
  const coverSizes = '(max-width: 640px) 100vw, (max-width: 1024px) 50vw, 25vw'

  function scrollSelectedIntoView() {
    // Wait for DOM to update with the selected card, then scroll into view within the pane
    tick().then(() => {
      if (!listEl || !selectedEpisode) return
      const key = `${selectedEpisode.id}:${selectedEpisode.episode}`
      const el = listEl.querySelector(`[data-ep-key="${key}"]`) as HTMLElement | null
      if (el) el.scrollIntoView({ behavior: 'smooth', block: 'nearest' })
    })
  }

  function onScrollPane() {
    if (localHighlightUntil > Date.now() && !shortenedOnce) {
      localHighlightUntil = Date.now() + 800
      shortenedOnce = true
    }
  }

  // Sync from parent only when the parent extends the highlight window
  $: if (highlightUntil > prevHighlightUntil) {
    prevHighlightUntil = highlightUntil
    localHighlightUntil = highlightUntil
    shortenedOnce = false
    scrollSelectedIntoView()
  }

  // Schedule local clear when time elapses to avoid indefinite pulse
  $: {
    if (localTimer) clearTimeout(localTimer)
    const ms = localHighlightUntil - Date.now()
    if (ms > 0) localTimer = setTimeout(() => { localHighlightUntil = 0 }, ms)
  }

  onDestroy(() => {
    if (localTimer) clearTimeout(localTimer)
  })
</script>

<div class="flex flex-col h-full rounded-lg overflow-hidden min-h-0">
  <div class={cn('flex-1 overflow-y-auto p-3 grid grid-cols-2', androidLandscape ? 'gap-2' : 'gap-3')} bind:this={listEl}
    style="content-visibility:auto; contain-intrinsic-size: 640px 400px;"
    on:scroll|passive={onScrollPane}>
    {#if episodes.length === 0}
      <div class="text-sm text-muted-foreground px-2 py-16 text-center">No episodes for this day</div>
    {:else}
      {#each episodes as ep (ep.id + ':' + ep.episode)}
        {@const status = listStatus(ep)}
        <ButtonPrimitive.Root href={'/app/anime/' + ep.id} data-ep-key={ep.id + ':' + ep.episode}
          class={cn(
            'w-full group rounded-md p-0 overflow-hidden border bg-background hover:bg-accent/90 text-left transition-transform transform-gpu duration-150 hover:scale-[1.03]',
            // Softer, lighter highlight; no white border; always pulse during active window
            selectedEpisode && ep.id === selectedEpisode.id && ep.episode === selectedEpisode.episode && localHighlightUntil > Date.now() && 'bg-accent/15 scale-[1.02] shadow-sm animate-pulse'
          )}
          on:click={() => onSelectEpisode(ep)}>
          {#if ep.bannerImage}
            <img src={ep.bannerImage} alt={ep.title?.userPreferred} class={cn('w-full object-cover', androidLandscape ? 'h-44' : 'h-48')} loading="lazy" decoding="async" sizes={coverSizes} />
          {:else if ep.coverImage?.extraLarge}
            <img src={ep.coverImage.extraLarge} alt={ep.title?.userPreferred} class={cn('w-full object-cover', androidLandscape ? 'h-44' : 'h-48')} loading="lazy" decoding="async" srcset={coverSrcset(ep.coverImage)} sizes={coverSizes} />
          {:else if ep.coverImage?.large}
            <img src={ep.coverImage.large} alt={ep.title?.userPreferred} class={cn('w-full object-cover', androidLandscape ? 'h-44' : 'h-48')} loading="lazy" decoding="async" srcset={coverSrcset(ep.coverImage)} sizes={coverSizes} />
          {:else if ep.coverImage?.medium}
            <img src={ep.coverImage.medium} alt={ep.title?.userPreferred} class={cn('w-full object-cover', androidLandscape ? 'h-44' : 'h-48')} loading="lazy" decoding="async" srcset={coverSrcset(ep.coverImage)} sizes={coverSizes} />
          {:else}
            <div class="w-full h-48 bg-muted" />
          {/if}
          <div class="p-3">
            <!-- Title row -->
            <div class="flex items-center gap-2">
              {#if status}
                <StatusDot variant={status} class="hidden xl:inline-flex" />
              {/if}
              <div class={cn('font-semibold text-ellipsis overflow-hidden whitespace-nowrap', androidLandscape ? 'text-sm' : '', +ep.airTime < Date.now() && 'line-through')} title={ep.title?.userPreferred}>{ep.title?.userPreferred}</div>
            </div>

            {#if wide1600}
              <!-- Wide (>=1600px): keep existing inline meta + chips -->
              <div class="mt-2 flex items-center justify-between gap-3">
                <div class="text-xs text-muted-foreground flex flex-wrap items-center gap-2">
                  {#if ep.format}<span>{ep.format}</span>{/if}
                  {#if ep.duration}<span>• Length: {ep.duration}m</span>{/if}
                  {#if ep.episodes}<span>• Episodes: {ep.episodes}</span>{/if}
                </div>
                <div class={cn('flex items-center gap-3', androidPortrait ? 'flex-wrap' : '')}>
                  <div class={cn('rounded-md bg-white/80 text-black border font-semibold', androidPortrait ? 'px-1.5 py-0.5 text-[12px]' : androidLandscape ? 'px-2 py-0.5 text-sm' : 'px-2 py-1 text-base md:text-lg')}>Ep {ep.episode}</div>
                  <div class={cn('rounded-md bg-white/80 text-black border font-semibold tabular-nums', androidPortrait ? 'px-1.5 py-0.5 text-[12px]' : androidLandscape ? 'px-2 py-0.5 text-sm' : 'px-2 py-1')}>
                    {format(ep.airTime, 'h:mm')} <span class="text-xs align-top uppercase">{format(ep.airTime, 'a')}</span>
                  </div>
                </div>
              </div>
            {:else}
              <!-- Compact (<1600px): meta on top (single line), chips below side-by-side -->
              <div class="mt-2 text-xs text-muted-foreground whitespace-nowrap overflow-hidden text-ellipsis">
                {#if ep.format}<span>{ep.format}</span>{/if}
                {#if ep.episodes}
                  <span class="ml-2">• {ep.episodes}eps</span>
                {/if}
                {#if ep.duration}
                  <span class="ml-2">• {ep.duration}m</span>
                {/if}
              </div>
              <div class={cn('mt-2 flex items-center gap-3', androidPortrait ? 'flex-wrap' : '')}>
                <div class={cn('rounded-md bg-white/80 text-black border font-semibold', androidPortrait ? 'px-1.5 py-0.5 text-[12px]' : androidLandscape ? 'px-2 py-0.5 text-sm' : 'px-2 py-1 text-base')}>Ep {ep.episode}</div>
                <div class={cn('rounded-md bg-white/80 text-black border font-semibold tabular-nums', androidPortrait ? 'px-1.5 py-0.5 text-[12px]' : androidLandscape ? 'px-2 py-0.5 text-sm' : 'px-2 py-1')}>
                  {format(ep.airTime, 'h:mm')} <span class="text-xs align-top uppercase">{format(ep.airTime, 'a')}</span>
                </div>
              </div>
            {/if}
          </div>
        </ButtonPrimitive.Root>
      {/each}
    {/if}
  </div>
</div>
