<script lang="ts">
  import { format } from 'date-fns'
  import { Button as ButtonPrimitive } from 'bits-ui'
  import { goto } from '$app/navigation'
  import { cn } from '$lib/utils'
  import { onMount, onDestroy, tick } from 'svelte'
  import { SUPPORTS } from '$lib/modules/settings'
  import StatusDot from '$lib/components/StatusDot.svelte'
  import { list as listStatus } from '$lib/modules/auth'

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
  export let selectedEpisode: any | undefined
  export let highlightUntil: number = 0

  let lastClickId: string | null = null
  let lastClickAt = 0

  function handleEpisodeClick(e: MouseEvent) {
    e.preventDefault()
    e.stopPropagation()
    const el = e.currentTarget as HTMLElement | null
    const idAttr = el?.dataset?.epId
    const epAttr = el?.dataset?.epEp
    const idNum = idAttr ? Number(idAttr) : NaN
    const epNum = epAttr ? Number(epAttr) : NaN
    const key = `${idNum}:${epNum}`

    // On Android, navigate immediately from the list no highlight
    if (SUPPORTS.isAndroid) { goto(`/app/anime/${idNum}`); return }

    // Find the episode object and highlight
    const findEp = (arr: any[]) => arr?.find((x) => x.id === idNum && x.episode === epNum)
    const epObj = findEp(currentEpisodes) || findEp(prevEpisodes) || findEp(nextEpisodes)

    const now = Date.now()
    if (lastClickId === key && (now - lastClickAt) <= 3000) {
      goto(`/app/anime/${idNum}`)
    } else {
      lastClickId = key
      lastClickAt = now
      if (epObj) onSelectEpisode(epObj)
    }
  }

  


  function selectPrevDate() { onSelectDate(prevDate) }
  function selectCurrentDate() { onSelectDate(currentDate) }
  function selectNextDate() { onSelectDate(nextDate) }

  function keydownSelectPrev(e: KeyboardEvent) { if (e.key === 'Enter' || e.key === ' ') { e.preventDefault(); e.stopPropagation(); onSelectDate(prevDate) } }
  function keydownSelectCurrent(e: KeyboardEvent) { if (e.key === 'Enter' || e.key === ' ') { e.preventDefault(); e.stopPropagation(); onSelectDate(currentDate) } }
  function keydownSelectNext(e: KeyboardEvent) { if (e.key === 'Enter' || e.key === ' ') { e.preventDefault(); e.stopPropagation(); onSelectDate(nextDate) } }

  // Android portrait: collapse current-day list to 6 items with More+ button
  let showAllCurrent = false
  $: visibleCurrentEpisodes = androidPortrait && !showAllCurrent ? currentEpisodes.slice(0, 6) : currentEpisodes

  let showAllPrev = false
  let showAllNext = false
  $: visiblePrevEpisodes = androidPortrait && !showAllPrev ? prevEpisodes.slice(0, 6) : prevEpisodes
  $: visibleNextEpisodes = androidPortrait && !showAllNext ? nextEpisodes.slice(0, 6) : nextEpisodes

  let wide88 = false
  let wide100 = false
  onMount(() => {
    const mq88 = window.matchMedia('(min-width: 88em)')
    const mq100 = window.matchMedia('(min-width: 100em)')
    const update = () => {
      wide88 = mq88.matches
      wide100 = mq100.matches
    }
    update()
    mq88.addEventListener('change', update)
    mq100.addEventListener('change', update)
    return () => {
      mq88.removeEventListener('change', update)
      mq100.removeEventListener('change', update)
    }
  })

  let _prevKey = 0, _currentKey = 0, _nextKey = 0
  $: { const k = +prevDate; if (k !== _prevKey) { _prevKey = k; showAllPrev = false } }
  $: { const k = +currentDate; if (k !== _currentKey) { _currentKey = k; showAllCurrent = false } }
  $: { const k = +nextDate; if (k !== _nextKey) { _nextKey = k; showAllNext = false } }

  // Detail pane utils (from DayDetailPane)
  let localHighlightUntil = 0
  let prevHighlightUntil = 0
  let localTimer: any
  let shortenedOnce = false
  let listEl: HTMLDivElement | null = null

  function scrollSelectedIntoView() {
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

  $: if (highlightUntil > prevHighlightUntil) {
    prevHighlightUntil = highlightUntil
    localHighlightUntil = highlightUntil
    shortenedOnce = false
    scrollSelectedIntoView()
  }

  $: {
    if (localTimer) clearTimeout(localTimer)
    const ms = localHighlightUntil - Date.now()
    if (ms > 0) localTimer = setTimeout(() => { localHighlightUntil = 0 }, ms)
  }

  onDestroy(() => { if (localTimer) clearTimeout(localTimer) })


  // Memoization caches
  type TimeParts = { t24: string, t12: string, am: string }
  const timeCache = new Map<number, TimeParts>()
  const timeKeys: number[] = []
  const TIME_CACHE_MAX = 200
  function getTimeParts(ts: any): TimeParts {
    const key = typeof ts === 'number' ? ts : +ts
    let e = timeCache.get(key)
    if (!e) {
      const d = new Date(key)
      e = { t24: format(d, 'HH:mm'), t12: format(d, 'h:mm'), am: format(d, 'a') }
      timeCache.set(key, e)
    }
    {
      const i = timeKeys.indexOf(key)
      if (i !== -1) timeKeys.splice(i, 1)
      timeKeys.push(key)
      pruneTimeCache()
    }
    return e
  }
  function time24(ts: any) { return getTimeParts(ts).t24 }
  function time12(ts: any) { return getTimeParts(ts).t12 }
  function ampm(ts: any) { return getTimeParts(ts).am }

  // Format release date (media startDate) with available granularity
  function releaseDate(ep: any) {
    const y = ep?.startDate?.year
    if (!y) return ''
    const m = ep?.startDate?.month
    const d = ep?.startDate?.day
    if (y && m && d) {
      const dt = new Date(y, m - 1, d)
      return format(dt, 'yyyy-MM-dd')
    }
    if (y && m) {
      const dt = new Date(y, m - 1, 1)
      return format(dt, 'yyyy-MM')
    }
    return String(y)
  }

  function dayStartMs(d: Date) { const x = new Date(d); x.setHours(0, 0, 0, 0); return +x }
  function isProtectedTime(t: number) {
    const day = 86400000
    const ps = dayStartMs(prevDate), cs = dayStartMs(currentDate), ns = dayStartMs(nextDate)
    return (t >= ps && t < ps + day) || (t >= cs && t < cs + day) || (t >= ns && t < ns + day)
  }
  function pruneTimeCache() {
    if (timeKeys.length <= TIME_CACHE_MAX) return
    let i = 0
    while (timeKeys.length > TIME_CACHE_MAX && i < timeKeys.length) {
      const k = timeKeys[i]!
      if (!isProtectedTime(k)) {
        timeKeys.splice(i, 1)
        timeCache.delete(k)
      } else {
        i++
      }
    }
  }

  const statusCache = new Map<string, any>()
  const statusKeys: string[] = []
  const STATUS_CACHE_MAX = 200
  function getStatus(ep: any) {
    const key = `${ep?.id}:${ep?.episode}`
    if (statusCache.has(key)) {
      const i = statusKeys.indexOf(key)
      if (i !== -1) statusKeys.splice(i, 1)
      statusKeys.push(key)
      return statusCache.get(key)
    }
    const v = listStatus(ep)
    statusCache.set(key, v)
    {
      const i = statusKeys.indexOf(key)
      if (i !== -1) statusKeys.splice(i, 1)
      statusKeys.push(key)
      pruneStatusCache()
    }
    return v
  }

  function pruneStatusCache() {
    if (statusKeys.length <= STATUS_CACHE_MAX) return
    const protectedKeys = new Set<string>()
    for (const e of prevEpisodes) protectedKeys.add(`${e?.id}:${e?.episode}`)
    for (const e of currentEpisodes) protectedKeys.add(`${e?.id}:${e?.episode}`)
    for (const e of nextEpisodes) protectedKeys.add(`${e?.id}:${e?.episode}`)
    let i = 0
    while (statusKeys.length > STATUS_CACHE_MAX && i < statusKeys.length) {
      const k = statusKeys[i]!
      if (!protectedKeys.has(k)) {
        statusKeys.splice(i, 1)
        statusCache.delete(k)
      } else {
        i++
      }
    }
  }



</script>

{#if androidPortrait}
  <div class='flex flex-col w-full gap-4 mx-auto'>
    <!-- Detail Pane -->
    <div class="flex flex-col h-full rounded-lg overflow-hidden min-h-0">
      <div class={cn('flex-1 overflow-y-auto p-3 grid grid-cols-2', androidLandscape ? 'gap-2' : 'gap-3')} bind:this={listEl}
        style="content-visibility:auto; contain-intrinsic-size: 640px 400px;"
        on:scroll={onScrollPane}>
        {#if currentEpisodes.length === 0}
          <div class="text-sm text-muted-foreground px-2 py-16 text-center">No episodes for this day</div>
        {:else}
          {#each currentEpisodes as ep (ep.id + ':' + ep.episode)}
            {@const status = getStatus(ep)}
            <ButtonPrimitive.Root href={'/app/anime/' + ep.id} data-ep-key={ep.id + ':' + ep.episode}
              class={cn('w-full group rounded-md p-0 overflow-hidden border bg-background hover:bg-accent/90 text-left transition-transform transform-gpu duration-150 hover:scale-[1.03]',
                         selectedEpisode && ep.id === selectedEpisode.id && ep.episode === selectedEpisode.episode && localHighlightUntil > Date.now() && 'bg-accent/15 scale-[1.02] shadow-sm animate-pulse')}
              on:click={() => onSelectEpisode(ep)}>
              {#if ep.bannerImage || ep.coverImage}
                {@const imgUrl = (ep.bannerImage || ep.coverImage?.extraLarge || ep.coverImage?.large || ep.coverImage?.medium)}
                <img src={imgUrl} alt={ep.title?.userPreferred} class={cn('w-full object-cover', androidLandscape ? 'h-44' : 'h-48')} loading="lazy" decoding="async" />
              {:else}
                <div class="w-full h-48 bg-muted" />
              {/if}
              <div class="p-3">
                <div class="flex items-center gap-2">
                  {#if status}
                    <StatusDot variant={status} class="hidden xl:inline-flex" />
                  {/if}
                  <div class={cn('font-semibold text-ellipsis overflow-hidden whitespace-nowrap', androidLandscape ? 'text-sm' : '', +ep.airTime < Date.now() && 'line-through')} title={ep.title?.userPreferred}>{ep.title?.userPreferred}</div>
                </div>
                {#if wide100}
                  <div class="mt-2 flex items-center justify-between gap-3">
                    <div class="text-xs text-muted-foreground flex flex-wrap items-center gap-2">
                      {#if ep.format}<span>{ep.format}</span>{/if}
                      {#if ep.duration}<span>• Length: {ep.duration}m</span>{/if}
                      {#if ep.episodes}<span>• Episodes: {ep.episodes}</span>{/if}
                      {#if ep.startDate?.year}<span>• Released: {releaseDate(ep)}</span>{/if}
                    </div>
                    <div class={cn('flex items-center gap-3', androidPortrait ? 'flex-wrap' : '')}>
                      <div class={cn('rounded-md bg-white/80 text-black border font-semibold', androidLandscape ? 'px-2 py-0.5 text-sm' : 'px-2 py-1 text-base md:text-lg')}>Ep {ep.episode}</div>
                      <div class={cn('rounded-md bg-white/80 text-black border font-semibold tabular-nums', androidLandscape ? 'px-2 py-0.5 text-sm' : 'px-2 py-1')}>
                        {time12(ep.airTime)} <span class="text-xs align-top uppercase">{ampm(ep.airTime)}</span>
                      </div>
                    </div>
                  </div>
                {:else}
                  <div class="mt-2 text-xs text-muted-foreground whitespace-nowrap overflow-hidden text-ellipsis">
                    {#if ep.format}<span>{ep.format}</span>{/if}
                    {#if ep.episodes}<span class="ml-2">• {ep.episodes}eps</span>{/if}
                    {#if ep.duration}<span class="ml-2">• {ep.duration}m</span>{/if}
                    {#if ep.startDate?.year}<span class="ml-2">• Released: {releaseDate(ep)}</span>{/if}
                  </div>
                  <div class={cn('mt-2 flex items-center gap-3', androidPortrait ? 'flex-wrap' : '')}>
                    <div class={cn('rounded-md bg-white/80 text-black border font-semibold', androidLandscape ? 'px-2 py-0.5 text-sm' : 'px-2 py-1 text-base')}>Ep {ep.episode}</div>
                    <div class={cn('rounded-md bg-white/80 text-black border font-semibold tabular-nums', androidLandscape ? 'px-2 py-0.5 text-sm' : 'px-2 py-1')}>
                      {time12(ep.airTime)} <span class="text-xs align-top uppercase">{ampm(ep.airTime)}</span>
                    </div>
                  </div>
                {/if}
              </div>
            </ButtonPrimitive.Root>
          {/each}
        {/if}
      </div>
    </div>

    <!-- Three-day lists -->
    <div class={cn(
      androidPortrait
        ? 'flex flex-col gap-3 h-full mt-2'
        : (wide88 ? (androidLandscape ? 'grid grid-cols-3 gap-2 h-full mt-2' : 'grid grid-cols-3 gap-3 h-full mt-2')
                    : (androidLandscape ? 'grid grid-cols-2 gap-2 h-full mt-2' : 'grid grid-cols-2 gap-3 h-full mt-2'))
    )}>
      {#if wide88}
      <div class={cn('flex flex-col border border-white/15 bg-muted/20 rounded-lg overflow-hidden hover:bg-accent/50', androidPortrait ? 'min-h-0 mt-0' : 'min-h-150 min-w-[13.5em] mt-2')}>
        <button type="button" class="px-3 py-3 text-neutral-200 text-base font-semibold bg-muted/60 cursor-pointer select-none text-center w-full" on:click={selectPrevDate} aria-label="Previous day">{format(prevDate, 'EEE d')}</button>
        <div class="flex-1 overflow-y-auto pt-2 px-1 pb-1 space-y-1.5 cursor-pointer" role="button" tabindex="0" style="content-visibility:auto; contain-intrinsic-size: 320px 400px;" on:click={selectPrevDate} on:keydown={keydownSelectPrev}>
          {#if prevEpisodes.length === 0}
            <div class="text-sm text-muted-foreground px-2 py-8 text-center">No episodes</div>
          {:else}
            {#each visiblePrevEpisodes as ep (ep.id + ':' + ep.episode)}
              <ButtonPrimitive.Root href={'/app/anime/' + ep.id} class={cn('flex items-center w-full text-neutral-200 group px-1 py-1 rounded-md')} on:click={handleEpisodeClick} data-ep-id={ep.id} data-ep-ep={ep.episode}>
                <div class={cn('w-12 pl-0.4 tabular-nums text-neutral-400 group-hover:text-neutral-100', androidPortrait ? 'text-[11px]' : androidLandscape ? 'text-[12px]' : 'text-xs')}>{time24(ep.airTime)}</div>
                <div class={cn('flex-1 pl-1.5 font-medium overflow-hidden', androidLandscape ? 'text-xs' : 'text-sm', +ep.airTime < Date.now() && 'line-through')} style="display:-webkit-box;-webkit-line-clamp:2;-webkit-box-orient:vertical;hyphens:auto;word-break:break-word;">{ep.title?.userPreferred}</div>
                <div class={cn('ml-auto pr-0.3 text-nowrap shrink-0', androidPortrait ? 'text-sm' : androidLandscape ? 'text-sm' : 'text-md')}>Ep {ep.episode}</div>
              </ButtonPrimitive.Root>
            {/each}
            {#if androidPortrait && prevEpisodes.length > 6 && !showAllPrev}
              <div class="flex justify-center py-1"><button type="button" class="px-3 py-1 rounded-md border text-sm opacity-90 hover:opacity-100" on:click={(e) => { e.preventDefault(); e.stopPropagation(); showAllPrev = true }}>More+</button></div>
            {/if}
          {/if}
        </div>
      </div>
      {/if}
      <div class={cn('flex flex-col border-2 border-white/25 bg-muted/40 rounded-lg overflow-hidden scale-[1.0005]', androidPortrait ? 'min-h-0' : 'min-h-0 min-w-[13.75em]')}>
        <button type="button" class="px-3 py-2.5 text-base font-semibold text-neutral-100 bg-muted/90 cursor-pointer select-none text-center w-full" on:click={selectCurrentDate} aria-label="Current day">{format(currentDate, 'EEE d')}</button>
        <div class="flex-1 overflow-y-auto pt-2 px-1 pb-1 space-y-1.5" style="content-visibility:auto; contain-intrinsic-size: 320px 400px;">
          {#if currentEpisodes.length === 0}
            <div class="text-sm text-muted-foreground px-2 py-8 text-center">No episodes</div>
          {:else}
            {#each visibleCurrentEpisodes as ep (ep.id + ':' + ep.episode)}
              <ButtonPrimitive.Root href={'/app/anime/' + ep.id} class={cn('flex items-center w-full group px-1 py-1 rounded-md')} on:click={handleEpisodeClick} data-ep-id={ep.id} data-ep-ep={ep.episode}>
                <div class={cn('w-12 pl-0.4 tabular-nums text-neutral-400 group-hover:text-neutral-200', androidPortrait ? 'text-[11px]' : androidLandscape ? 'text-[12px]' : 'text-xs md:text-sm')}>{time24(ep.airTime)}</div>
                <div class={cn('flex-1 pl-1.5 font-medium overflow-hidden', androidLandscape ? 'text-xs' : 'text-sm', +ep.airTime < Date.now() && 'line-through')} style="display:-webkit-box;-webkit-line-clamp:2;-webkit-box-orient:vertical;hyphens:auto;word-break:break-word;">{ep.title?.userPreferred}</div>
                <div class={cn('ml-auto pr-0.3 text-nowrap shrink-0', androidPortrait ? 'text-sm' : androidLandscape ? 'text-sm' : 'md:text-base')}>Ep {ep.episode}</div>
              </ButtonPrimitive.Root>
            {/each}
            {#if androidPortrait && currentEpisodes.length > 6 && !showAllCurrent}
              <div class="flex justify-center py-1"><button type="button" class="px-3 py-1 rounded-md border text-sm opacity-90 hover:opacity-100" on:click={() => showAllCurrent = true}>More+</button></div>
            {/if}
          {/if}
        </div>
      </div>
      <div class={cn('flex flex-col border border-white/15 bg-muted/20 rounded-lg overflow-hidden hover:bg-accent/25', androidPortrait ? 'min-h-0 mt-0' : 'min-h-0 min-w-[13.5em] mt-2')}>
        <button type="button" class="px-3 py-3 text-base text-neutral-200 font-semibold bg-muted/60 cursor-pointer select-none text-center w-full" on:click={selectNextDate} aria-label="Next day">{format(nextDate, 'EEE d')}</button>
        <div class="flex-1 overflow-y-auto pt-2 px-1 pb-1 space-y-1.5 cursor-pointer" role="button" tabindex="0" style="content-visibility:auto; contain-intrinsic-size: 320px 400px;" on:click={selectNextDate} on:keydown={keydownSelectNext}>
          {#if nextEpisodes.length === 0}
            <div class="text-sm text-muted-foreground px-2 py-8 text-center">No episodes</div>
          {:else}
            {#each visibleNextEpisodes as ep (ep.id + ':' + ep.episode)}
              <ButtonPrimitive.Root href={'/app/anime/' + ep.id} class={cn('flex items-center text-neutral-200 w-full group px-1 py-1 rounded-md')} on:click={handleEpisodeClick} data-ep-id={ep.id} data-ep-ep={ep.episode}>
                <div class={cn('w-12 pl-0.4 tabular-nums text-neutral-400 group-hover:text-neutral-200', androidPortrait ? 'text-[11px]' : androidLandscape ? 'text-[12px]' : 'text-xs')}>{time24(ep.airTime)}</div>
                <div class={cn('flex-1 pl-1.5 font-medium overflow-hidden', androidLandscape ? 'text-xs' : 'text-sm', +ep.airTime < Date.now() && 'line-through')} style="display:-webkit-box;-webkit-line-clamp:2;-webkit-box-orient:vertical;hyphens:auto;word-break:break-word;">{ep.title?.userPreferred}</div>
                <div class={cn('ml-auto pr-0.3 text-nowrap shrink-0', androidPortrait ? 'text-sm' : androidLandscape ? 'text-sm' : 'text-md')}>Ep {ep.episode}</div>
              </ButtonPrimitive.Root>
            {/each}
            {#if androidPortrait && nextEpisodes.length > 6 && !showAllNext}
              <div class="flex justify-center py-1"><button type="button" class="px-3 py-1 rounded-md border text-sm opacity-90 hover:opacity-100" on:click={(e) => { e.preventDefault(); e.stopPropagation(); showAllNext = true }}>More+</button></div>
            {/if}
          {/if}
        </div>
      </div>
    </div>
  </div>
{:else}
  <div class='grid w-full gap-4 mx-auto min-w-[62.5em]' style='grid-template-columns: 1fr 1fr;'>
    <!-- Lists -->
    <div class={cn(
      wide88 ? (androidLandscape ? 'grid grid-cols-3 gap-2 h-full mt-2' : 'grid grid-cols-3 gap-3 h-full mt-2')
               : (androidLandscape ? 'grid grid-cols-2 gap-2 h-full mt-2' : 'grid grid-cols-2 gap-3 h-full mt-2')
    )}>
      {#if wide88}
      <div class={cn('flex flex-col border border-white/15 bg-muted/20 rounded-lg overflow-hidden hover:bg-accent/50', 'min-h-150 min-w-[13.5em] mt-2')}>
        <button type="button" class="px-3 py-3 text-neutral-200 text-base font-semibold bg-muted/60 cursor-pointer select-none text-center w-full" on:click={selectPrevDate} aria-label="Previous day">{format(prevDate, 'EEE d')}</button>
        <div class="flex-1 overflow-y-auto pt-2 px-1 pb-1 space-y-1.5 cursor-pointer" role="button" tabindex="0" style="content-visibility:auto; contain-intrinsic-size: 320px 400px;" on:click={selectPrevDate} on:keydown={keydownSelectPrev}>
          {#if prevEpisodes.length === 0}
            <div class="text-sm text-muted-foreground px-2 py-8 text-center">No episodes</div>
          {:else}
            {#each visiblePrevEpisodes as ep (ep.id + ':' + ep.episode)}
              <ButtonPrimitive.Root href={'/app/anime/' + ep.id} class={cn('flex items-center w-full text-neutral-200 group px-1 py-1 rounded-md')} on:click={handleEpisodeClick} data-ep-id={ep.id} data-ep-ep={ep.episode}>
                <div class={cn('w-12 pl-0.4 tabular-nums text-neutral-400 group-hover:text-neutral-100', androidLandscape ? 'text-[12px]' : 'text-xs')}>{time24(ep.airTime)}</div>
                <div class={cn('flex-1 pl-1.5 font-medium overflow-hidden', androidLandscape ? 'text-xs' : 'text-sm', +ep.airTime < Date.now() && 'line-through')} style="display:-webkit-box;-webkit-line-clamp:2;-webkit-box-orient:vertical;hyphens:auto;word-break:break-word;">{ep.title?.userPreferred}</div>
                <div class={cn('ml-auto pr-0.3 text-nowrap shrink-0', androidLandscape ? 'text-sm' : 'text-md')}>Ep {ep.episode}</div>
              </ButtonPrimitive.Root>
            {/each}
          {/if}
        </div>
      </div>
      {/if}
      <div class={cn('flex flex-col border-2 border-white/25 bg-muted/40 rounded-lg overflow-hidden scale-[1.0005]', 'min-h-0 min-w-[13.75em]')}>
        <button type="button" class="px-3 py-2.5 text-base font-semibold text-neutral-100 bg-muted/90 cursor-pointer select-none text-center w-full" on:click={selectCurrentDate} aria-label="Current day">{format(currentDate, 'EEE d')}</button>
        <div class="flex-1 overflow-y-auto pt-2 px-1 pb-1 space-y-1.5" style="content-visibility:auto; contain-intrinsic-size: 320px 400px;">
          {#if currentEpisodes.length === 0}
            <div class="text-sm text-muted-foreground px-2 py-8 text-center">No episodes</div>
          {:else}
            {#each visibleCurrentEpisodes as ep (ep.id + ':' + ep.episode)}
              <ButtonPrimitive.Root href={'/app/anime/' + ep.id} class={cn('flex items-center w-full group px-1 py-1 rounded-md')} on:click={handleEpisodeClick} data-ep-id={ep.id} data-ep-ep={ep.episode}>
                <div class={cn('w-12 pl-0.4 tabular-nums text-neutral-400 group-hover:text-neutral-200', androidLandscape ? 'text-[12px]' : 'text-xs md:text-sm')}>{time24(ep.airTime)}</div>
                <div class={cn('flex-1 pl-1.5 font-medium overflow-hidden', androidLandscape ? 'text-xs' : 'text-sm', +ep.airTime < Date.now() && 'line-through')} style="display:-webkit-box;-webkit-line-clamp:2;-webkit-box-orient:vertical;hyphens:auto;word-break:break-word;">{ep.title?.userPreferred}</div>
                <div class={cn('ml-auto pr-0.3 text-nowrap shrink-0', androidLandscape ? 'text-sm' : 'md:text-base')}>Ep {ep.episode}</div>
              </ButtonPrimitive.Root>
            {/each}
          {/if}
        </div>
      </div>
      <div class={cn('flex flex-col border border-white/15 bg-muted/20 rounded-lg overflow-hidden hover:bg-accent/25', 'min-h-0 min-w-[13.5em] mt-2')}>
        <button type="button" class="px-3 py-3 text-base text-neutral-200 font-semibold bg-muted/60 cursor-pointer select-none text-center w-full" on:click={selectNextDate} aria-label="Next day">{format(nextDate, 'EEE d')}</button>
        <div class="flex-1 overflow-y-auto pt-2 px-1 pb-1 space-y-1.5 cursor-pointer" role="button" tabindex="0" style="content-visibility:auto; contain-intrinsic-size: 320px 400px;" on:click={selectNextDate} on:keydown={keydownSelectNext}>
          {#if nextEpisodes.length === 0}
            <div class="text-sm text-muted-foreground px-2 py-8 text-center">No episodes</div>
          {:else}
            {#each visibleNextEpisodes as ep (ep.id + ':' + ep.episode)}
              <ButtonPrimitive.Root href={'/app/anime/' + ep.id} class={cn('flex items-center text-neutral-200 w-full group px-1 py-1 rounded-md')} on:click={handleEpisodeClick} data-ep-id={ep.id} data-ep-ep={ep.episode}>
                <div class={cn('w-12 pl-0.4 tabular-nums text-neutral-400 group-hover:text-neutral-200', androidLandscape ? 'text-[12px]' : 'text-xs')}>{time24(ep.airTime)}</div>
                <div class={cn('flex-1 pl-1.5 font-medium overflow-hidden', androidLandscape ? 'text-xs' : 'text-sm', +ep.airTime < Date.now() && 'line-through')} style="display:-webkit-box;-webkit-line-clamp:2;-webkit-box-orient:vertical;hyphens:auto;word-break:break-word;">{ep.title?.userPreferred}</div>
                <div class={cn('ml-auto pr-0.3 text-nowrap shrink-0', androidLandscape ? 'text-sm' : 'text-md')}>Ep {ep.episode}</div>
              </ButtonPrimitive.Root>
            {/each}
          {/if}
        </div>
      </div>
    </div>

    <!-- Detail Pane -->
    <div class="flex flex-col h-full rounded-lg overflow-hidden min-h-0">
      <div class={cn('flex-1 overflow-y-auto p-3 grid grid-cols-2', androidLandscape ? 'gap-2' : 'gap-3')} bind:this={listEl}
        style="content-visibility:auto; contain-intrinsic-size: 640px 400px;"
        on:scroll={onScrollPane}>
        {#if currentEpisodes.length === 0}
          <div class="text-sm text-muted-foreground px-2 py-16 text-center">No episodes for this day</div>
        {:else}
          {#each currentEpisodes as ep (ep.id + ':' + ep.episode)}
            {@const status = getStatus(ep)}
            <ButtonPrimitive.Root href={'/app/anime/' + ep.id} data-ep-key={ep.id + ':' + ep.episode}
              class={cn('w-full group rounded-md p-0 overflow-hidden border bg-background hover:bg-accent/90 text-left transition-transform transform-gpu duration-150 hover:scale-[1.03]',
                         selectedEpisode && ep.id === selectedEpisode.id && ep.episode === selectedEpisode.episode && localHighlightUntil > Date.now() && 'bg-accent/15 scale-[1.02] shadow-sm animate-pulse')}
              on:click={() => onSelectEpisode(ep)}>
              {#if ep.bannerImage || ep.coverImage}
                {@const imgUrl = (ep.bannerImage || ep.coverImage?.extraLarge || ep.coverImage?.large || ep.coverImage?.medium)}
                <img src={imgUrl} alt={ep.title?.userPreferred} class={cn('w-full object-cover', androidLandscape ? 'h-44' : 'h-48')} loading="lazy" decoding="async" />
              {:else}
                <div class="w-full h-48 bg-muted" />
              {/if}
              <div class="p-3">
                <div class="flex items-center gap-2">
                  {#if status}
                    <StatusDot variant={status} class="hidden xl:inline-flex" />
                  {/if}
                  <div class={cn('font-semibold text-ellipsis overflow-hidden whitespace-nowrap', androidLandscape ? 'text-sm' : '', +ep.airTime < Date.now() && 'line-through')} title={ep.title?.userPreferred}>{ep.title?.userPreferred}</div>
                </div>
                {#if wide100}
                  <div class="mt-2 flex items-center justify-between gap-3">
                    <div class="text-xs text-muted-foreground flex flex-wrap items-center gap-2">
                      {#if ep.format}<span>{ep.format}</span>{/if}
                      {#if ep.duration}<span>• Length: {ep.duration}m</span>{/if}
                      {#if ep.episodes}<span>• Episodes: {ep.episodes}</span>{/if}
                      {#if ep.startDate?.year}<span>• Released: {releaseDate(ep)}</span>{/if}
                    </div>
                    <div class={cn('flex items-center gap-3', androidPortrait ? 'flex-wrap' : '')}>
                      <div class={cn('rounded-md bg-white/80 text-black border font-semibold', androidLandscape ? 'px-2 py-0.5 text-sm' : 'px-2 py-1 text-base md:text-lg')}>Ep {ep.episode}</div>
                      <div class={cn('rounded-md bg-white/80 text-black border font-semibold tabular-nums', androidLandscape ? 'px-2 py-0.5 text-sm' : 'px-2 py-1')}>
                        {time12(ep.airTime)} <span class="text-xs align-top uppercase">{ampm(ep.airTime)}</span>
                      </div>
                    </div>
                  </div>
                {:else}
                  <div class="mt-2 text-xs text-muted-foreground whitespace-nowrap overflow-hidden text-ellipsis">
                    {#if ep.format}<span>{ep.format}</span>{/if}
                    {#if ep.episodes}<span class="ml-2">• {ep.episodes}eps</span>{/if}
                    {#if ep.duration}<span class="ml-2">• {ep.duration}m</span>{/if}
                    {#if ep.startDate?.year}<span class="ml-2">• Released: {releaseDate(ep)}</span>{/if}
                  </div>
                  <div class={cn('mt-2 flex items-center gap-3', androidPortrait ? 'flex-wrap' : '')}>
                    <div class={cn('rounded-md bg-white/80 text-black border font-semibold', androidLandscape ? 'px-2 py-0.5 text-sm' : 'px-2 py-1 text-base')}>Ep {ep.episode}</div>
                    <div class={cn('rounded-md bg-white/80 text-black border font-semibold tabular-nums', androidLandscape ? 'px-2 py-0.5 text-sm' : 'px-2 py-1')}>
                      {time12(ep.airTime)} <span class="text-xs align-top uppercase">{ampm(ep.airTime)}</span>
                    </div>
                  </div>
                {/if}
              </div>
            </ButtonPrimitive.Root>
          {/each}
        {/if}
      </div>
    </div>
  </div>
{/if}
