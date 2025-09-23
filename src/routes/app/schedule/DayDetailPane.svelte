<script lang="ts">
  import { format } from 'date-fns'
  import { Button as ButtonPrimitive } from 'bits-ui'
  import { onDestroy, tick } from 'svelte'
  import StatusDot from '$lib/components/StatusDot.svelte'
  import { list as listStatus } from '$lib/modules/auth'
  import { cn } from '$lib/utils'

  export let episodes: any[] = []
  export let selectedEpisode: any | undefined
  export let onSelectEpisode: (ep: any) => void
  export let highlightUntil: number = 0
  let localHighlightUntil = 0
  let prevHighlightUntil = 0
  let localTimer: any
  let shortenedOnce = false
  let listEl: HTMLDivElement | null = null

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
  <div class="flex-1 overflow-y-auto p-3 grid grid-cols-2 gap-3" bind:this={listEl}
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
            <img src={ep.bannerImage} alt={ep.title?.userPreferred} class="w-full h-48 object-cover" loading="lazy" decoding="async" sizes={coverSizes} />
          {:else if ep.coverImage?.extraLarge}
            <img src={ep.coverImage.extraLarge} alt={ep.title?.userPreferred} class="w-full h-48 object-cover" loading="lazy" decoding="async" srcset={coverSrcset(ep.coverImage)} sizes={coverSizes} />
          {:else if ep.coverImage?.large}
            <img src={ep.coverImage.large} alt={ep.title?.userPreferred} class="w-full h-48 object-cover" loading="lazy" decoding="async" srcset={coverSrcset(ep.coverImage)} sizes={coverSizes} />
          {:else if ep.coverImage?.medium}
            <img src={ep.coverImage.medium} alt={ep.title?.userPreferred} class="w-full h-48 object-cover" loading="lazy" decoding="async" srcset={coverSrcset(ep.coverImage)} sizes={coverSizes} />
          {:else}
            <div class="w-full h-48 bg-muted" />
          {/if}
          <div class="p-3">
            <div class="flex items-center gap-2">
              {#if status}
                <StatusDot variant={status} class="hidden xl:inline-flex" />
              {/if}
              <div class={cn('font-semibold text-ellipsis overflow-hidden whitespace-nowrap', +ep.airTime < Date.now() && 'line-through')} title={ep.title?.userPreferred}>{ep.title?.userPreferred}</div>
            </div>
            <div class="mt-2 flex items-center justify-between gap-3">
              <div class="text-xs text-muted-foreground flex flex-wrap items-center gap-2">
                {#if ep.format}<span>{ep.format}</span>{/if}
                {#if ep.duration}<span>• Length: {ep.duration}m</span>{/if}
                {#if ep.episodes}<span>• Episodes: {ep.episodes}</span>{/if}
              </div>
              <div class="flex items-center gap-3">
                <div class="px-2 py-1 rounded-md bg-white/80 text-black border text-base md:text-lg font-semibold whitespace-nowrap">Ep {ep.episode}</div>
                <div class="px-2 py-1 rounded-md bg-white/80 text-black border font-semibold tabular-nums whitespace-nowrap">
                  {format(ep.airTime, 'h:mm')} <span class="text-xs align-top uppercase">{format(ep.airTime, 'a')}</span>
                </div>
              </div>
            </div>
          </div>
        </ButtonPrimitive.Root>
      {/each}
    {/if}
  </div>
</div>
