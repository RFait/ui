<script lang='ts'>
  import { Button as ButtonPrimitive } from 'bits-ui'
  import { addMonths, endOfMonth, endOfWeek, format, isSameMonth, isToday, startOfMonth, startOfWeek, subMonths } from 'date-fns'
  import { persisted } from 'svelte-persisted-store'
  import Cross2 from 'svelte-radix/Cross2.svelte'
  import { onMount } from 'svelte'

  import type { Schedule, ScheduleMedia } from '$lib/modules/anilist/queries'
  import type { ResultOf } from 'gql.tada'

  import StatusDot from '$lib/components/StatusDot.svelte'
  import { ChevronLeft, ChevronRight } from '$lib/components/icons/animated'
  import { Button } from '$lib/components/ui/button'
  import * as Drawer from '$lib/components/ui/drawer'
  import { Label } from '$lib/components/ui/label'
  import { Switch } from '$lib/components/ui/switch'
  import * as Tooltip from '$lib/components/ui/tooltip'
  import { dedupeAiring } from '$lib/modules/anilist'
  import { authAggregator, list } from '$lib/modules/auth'
  import { dragScroll } from '$lib/modules/navigate'
  import { cn, breakpoints } from '$lib/utils'
  import { SUPPORTS } from '$lib/modules/settings'
  import ThreeDayView from './ThreeDayView.svelte'
  import DayDetailPane from './DayDetailPane.svelte'

  const onList = persisted('schedule-on-list', true)
  const viewMode = persisted<'three-day' | 'month'>('schedule-view-mode', 'three-day')

  $: query = authAggregator.schedule($onList || null)

  // Month grid state (existing view)
  let now = new Date()

  $: firstDay = startOfWeek(startOfMonth(now), { weekStartsOn: 1 })
  $: lastDay = endOfWeek(endOfMonth(now), { weekStartsOn: 1 })
  function prevMonth () {
    now = subMonths(now, 1)
    selectedDate = subMonths(selectedDate, 1)
  }
  function nextMonth () {
    now = addMonths(now, 1)
    selectedDate = addMonths(selectedDate, 1)
  }

  function listDays (firstDay: Date, lastDay: Date) {
    // create an array of days with start and end time for given day
    const days = []
    // eslint-disable-next-line no-unmodified-loop-condition
    for (let start = new Date(firstDay); start <= lastDay; start.setDate(start.getDate() + 1)) {
      days.push({ date: new Date(start), number: start.getDate() })
    }
    return days
  }

  $: dayList = listDays(firstDay, lastDay)

  interface DayAirTimes { day: { date: Date, number: number }, episodes: Array<ResultOf<typeof ScheduleMedia> & { episode: number, airTime: Date }> }

  function aggregate (data: ResultOf<typeof Schedule>, dayList: Array<{ date: Date, number: number }>) {
    // join media from all queries into single list, de-duplicate it, and make sure it's not dropped
    const mediaList = [...data.curr1?.media ?? [], ...data.curr2?.media ?? [], ...data.curr3?.media ?? [], ...data.residue?.media ?? [], ...data.next1?.media ?? [], ...data.next2?.media ?? []]
      .filter((v, i, a) => v != null
        && a.findIndex(s => s?.id === v.id) === i
        && list(v) !== 'DROPPED'
        && (!$onList || (v as any)?.mediaListEntry)) as Array<ResultOf<typeof ScheduleMedia>>

    const dayMap: Record<string, DayAirTimes | undefined> = Object.fromEntries(dayList.map(day => [+day.date, { day, episodes: [] }]))

    for (const media of mediaList) {
      // dedupe airing lists
      const episodes = dedupeAiring(media)
      for (const { a: airingAt, e: episode } of episodes) {
        const airTime = new Date(airingAt * 1000)
        airTime.setHours(0, 0, 0, 0)
        const day = dayMap[+airTime]
        if (day) day.episodes.push({ ...media, episode, airTime: new Date(airingAt * 1000) })
      }
    }

    for (const { episodes } of Object.values(dayMap) as DayAirTimes[]) {
      episodes.sort((a, b) => +a.airTime - +b.airTime)
    }

    return Object.values(dayMap) as DayAirTimes[]
  }

  // very stupid fix, for a very stupid bug
  const _list = list

  // Three-day view state
  let selectedDate = new Date()
  function midnight (d: Date) {
    const x = new Date(d)
    x.setHours(0, 0, 0, 0)
    return x
  }
  function prevDay () { selectedDate = new Date(selectedDate.getFullYear(), selectedDate.getMonth(), selectedDate.getDate() - 1) }
  function nextDay () { selectedDate = new Date(selectedDate.getFullYear(), selectedDate.getMonth(), selectedDate.getDate() + 1) }
  function goToday () { selectedDate = new Date() }

  function cycleViewMode() {
    $viewMode = $viewMode === 'three-day' ? 'month' : 'three-day'
  }

  type EpisodeItem = ResultOf<typeof ScheduleMedia> & { episode: number, airTime: Date }

  function groupByDay (data: ResultOf<typeof Schedule> | undefined) {
    const map: Record<string, { date: Date, episodes: EpisodeItem[] }> = {}
    if (!data) return map

    const mediaList = [
      ...data.curr1?.media ?? [],
      ...data.curr2?.media ?? [],
      ...data.curr3?.media ?? [],
      ...data.residue?.media ?? [],
      ...data.next1?.media ?? [],
      ...data.next2?.media ?? []
    ].filter((v, i, a) => v != null
      && a.findIndex(s => s?.id === v.id) === i
      && list(v as ResultOf<typeof ScheduleMedia>) !== 'DROPPED'
      && (!$onList || (v as any)?.mediaListEntry)) as Array<ResultOf<typeof ScheduleMedia>>

    for (const media of mediaList) {
      const episodes = dedupeAiring(media)
      for (const { a: airingAt, e: episode } of episodes) {
        const dt = new Date(airingAt * 1000)
        const key = +midnight(dt)
        if (!map[key]) map[key] = { date: midnight(dt), episodes: [] }
        map[key].episodes.push({ ...media, episode, airTime: dt })
      }
    }
    for (const day of Object.values(map)) day.episodes.sort((a, b) => +a.airTime - +b.airTime)
    return map
  }

  $: grouped = groupByDay($query.data)
  $: prevDate = new Date(selectedDate.getFullYear(), selectedDate.getMonth(), selectedDate.getDate() - 1)
  $: nextDate = new Date(selectedDate.getFullYear(), selectedDate.getMonth(), selectedDate.getDate() + 1)
  $: prevEpisodes = grouped[+midnight(prevDate)]?.episodes ?? []
  $: currentEpisodes = grouped[+midnight(selectedDate)]?.episodes ?? []
  $: nextEpisodes = grouped[+midnight(nextDate)]?.episodes ?? []

  let selectedEpisode: EpisodeItem | undefined
  let highlightUntil = 0
  let highlightTimer: any

  function highlightEpisode(ep: EpisodeItem) {
    selectedEpisode = ep
    selectedDate = midnight(ep.airTime)
    if (highlightTimer) clearTimeout(highlightTimer)
    highlightUntil = Date.now() + 3000
    highlightTimer = setTimeout(() => { highlightUntil = 0 }, 3000)
  }

  // Android portrait detection
  let androidPortrait = false
  let androidLandscape = false
  onMount(() => {
    const mq = window.matchMedia('(orientation: portrait)')
    const update = () => {
      androidPortrait = SUPPORTS.isAndroid && mq.matches
      androidLandscape = SUPPORTS.isAndroid && !mq.matches
    }
    update()
    mq.addEventListener('change', update)
    return () => {
      mq.removeEventListener('change', update)
    }
  })

  // Today state for button styling
  $: isOnToday = +midnight(selectedDate) === +midnight(new Date())
</script>

<div class='w-full h-full overflow-auto p-2 md:p-6 min-w-0' use:dragScroll style='scrollbar-gutter: stable both-edges;'>
  <div class='space-y-2 mb-4 w-full max-w-none px-2 md:px-4 mx-auto'>
    <div class='flex items-center justify-between'>
      <div>
        <h2 class='text-2xl font-bold'>Airing Calendar</h2>
        <p class='text-muted-foreground'>Find what’s airing now and next.</p>
      </div>
      <!-- View mode buttons moved to the main title bar below -->
    </div>

    <!-- Title bar (Android portrait specific vs default) -->
    {#if androidPortrait}
      <!-- Android portrait: top row with month/day and arrows; second row with My list, Today, and View toggle -->
      <div class='w-full flex flex-col gap-2'>
        <!-- Top: Month/Day + arrows (stable positions) -->
        <div class='w-full grid grid-cols-[auto_1fr_auto] items-center gap-1.5'>
          <div class='flex items-center gap-1.5 justify-self-start'>
            <Button size='icon' on:click={prevDay} variant='outline' class='bg-transparent animated-icon h-7 w-7'>
              <ChevronLeft class='h-5 w-5' />
            </Button>
            <Button size='icon' on:click={prevMonth} variant='outline' class='bg-transparent animated-icon h-7 w-7'>
              <ChevronLeft class='h-5 w-5' />
            </Button>
          </div>
          <div class='text-base font-semibold px-1 text-center justify-self-center min-w-[18ch] max-w-[60vw] truncate'>
            {format(selectedDate, 'MMMM, EEEE do')}{selectedDate.getFullYear() !== new Date().getFullYear() ? ` ${format(selectedDate, 'yyyy')}` : ''}
          </div>
          <div class='flex items-center gap-1.5 justify-self-end'>
            <Button size='icon' on:click={nextMonth} variant='outline' class='bg-transparent animated-icon h-7 w-7'>
              <ChevronRight class='h-5 w-5' />
            </Button>
            <Button size='icon' on:click={nextDay} variant='outline' class='bg-transparent animated-icon h-7 w-7'>
              <ChevronRight class='h-5 w-5' />
            </Button>
          </div>
        </div>
        <!-- Bottom: My list, Today, View toggle -->
        <div class='w-full grid grid-cols-3 items-center gap-2'>
          <div class='flex items-center gap-2 text-muted-foreground justify-self-start'>
            <Switch bind:checked={$onList} id='schedule-on-list' hideState={true} />
            <Label for='schedule-on-list'>My list</Label>
          </div>
          <div class='flex items-center justify-center'>
            <Button size='sm' variant='outline' class={isOnToday ? 'border-white/70 bg-white/10 text-white' : 'border-white/30 text-white/90 hover:bg-white/5'} on:click={goToday}>Today</Button>
          </div>
          <div class='flex items-center gap-2 justify-self-end'>
            <Button size='sm' variant='outline' class='w-[96px] justify-center' on:click={cycleViewMode}>
              {$viewMode === 'three-day' ? 'Three-day' : 'Month'}
            </Button>
          </div>
        </div>
      </div>
    {:else}
      <!-- Default (non-Android or landscape): left My list, center Month/Day + arrows + Today, right View toggle -->
      <div class='grid items-center w-full grid-cols-[1fr_auto_1fr]'>
        <!-- My list -->
        <div class='flex items-center gap-1 text-muted-foreground justify-self-start'>
          <Switch bind:checked={$onList} id='schedule-on-list' hideState={true} />
          <Label for='schedule-on-list'>My list</Label>
        </div>

        <!--  Day and Month container (stable positions) -->
        <div class='justify-self-center grid grid-cols-[auto_1fr_auto] items-center gap-1.5'>
          <div class='flex items-center gap-1.5 justify-self-start'>
            <Button size='icon' on:click={prevDay} variant='outline' class='bg-transparent animated-icon h-7 w-7'>
              <ChevronLeft class='h-5 w-5' />
            </Button>
            <Button size='icon' on:click={prevMonth} variant='outline' class='bg-transparent animated-icon h-9 w-9'>
              <ChevronLeft class='h-6 w-6' />
            </Button>
          </div>
          <div class='text-xl font-semibold px-1 text-center justify-self-center min-w-[24ch]'>
            {format(selectedDate, 'MMMM, EEEE do')}{selectedDate.getFullYear() !== new Date().getFullYear() ? ` ${format(selectedDate, 'yyyy')}` : ''}
          </div>
          <div class='flex items-center gap-1.5 justify-self-end'>
            <Button size='icon' on:click={nextMonth} variant='outline' class='bg-transparent animated-icon h-9 w-9'>
              <ChevronRight class='h-6 w-6' />
            </Button>
            <Button size='icon' on:click={nextDay} variant='outline' class='bg-transparent animated-icon h-7 w-7'>
              <ChevronRight class='h-5 w-5' />
            </Button>
          </div>
          <!-- Today button remains on the right grouping in default layout -->
          <Button class={'ml-2 col-span-3 justify-self-center hidden md:inline-flex ' + (isOnToday ? 'border-white/70 bg-white/10 text-white' : 'border-white/30 text-white/90 hover:bg-white/5')} size='sm' variant='outline' on:click={goToday}>Today</Button>
        </div>

        <!-- View mode toggle -->
        <div class='flex items-center justify-self-end'>
          <Button size='sm' variant='outline' class='w-[96px] justify-center' on:click={cycleViewMode}>
            {$viewMode === 'three-day' ? 'Three-day' : 'Month'}
          </Button>
        </div>
      </div>
    {/if}
  </div>

  {#if $query.fetching}
    <div class='p-5 flex items-center justify-center h-96 w-full max-w-[1800px]'>Loading…</div>
  {:else if $query.error}
    <div class='p-5 flex items-center justify-center h-96 w-full max-w-[1800px]'>
      <div>
        <div class='mb-1 font-bold text-4xl text-center '>
          Ooops!
        </div>
        <div class='text-lg text-center text-muted-foreground'>
          Looks like something went wrong!
        </div>
        <div class='text-lg text-center text-muted-foreground'>
          {$query.error.message}
        </div>
      </div>
    </div>
  {:else}
    {#if $viewMode === 'three-day'}
      {#if androidPortrait}
        <!-- Android portrait: stack thumbnails (detail pane) on top, list (three-day) under -->
        <div class='flex flex-col w-full gap-4 mx-auto'>
          <DayDetailPane episodes={currentEpisodes} {selectedEpisode} {highlightUntil} onSelectEpisode={(ep) => selectedEpisode = ep} androidPortrait={true} />
          <ThreeDayView
            {prevDate}
            currentDate={selectedDate}
            {nextDate}
            {prevEpisodes}
            {currentEpisodes}
            {nextEpisodes}
            onSelectDate={(d) => selectedDate = d}
            onSelectEpisode={highlightEpisode}
            androidPortrait={true}
          />
        </div>
      {:else}
        <div class='grid w-full gap-4 mx-auto min-w-[1000px]' style='grid-template-columns: 1fr 1fr;'>
          <ThreeDayView
            {prevDate}
            currentDate={selectedDate}
            {nextDate}
            {prevEpisodes}
            {currentEpisodes}
            {nextEpisodes}
            onSelectDate={(d) => selectedDate = d}
            onSelectEpisode={highlightEpisode}
            androidLandscape={androidLandscape}
          />
          <DayDetailPane episodes={currentEpisodes} {selectedEpisode} {highlightUntil} onSelectEpisode={(ep) => selectedEpisode = ep} androidLandscape={androidLandscape} />
        </div>
      {/if}
    {:else if $viewMode === 'month'}
      <div class='grid grid-cols-7 border rounded-lg [&>*:not(:nth-child(7n+1))]:border-l [&>*:nth-last-child(n+8)]:border-b [&>*:nth-child(-n+7)]:border-b w-full mx-auto'>
        <div class='text-center py-2'>Mon</div>
        <div class='text-center py-2'>Tue</div>
        <div class='text-center py-2'>Wed</div>
        <div class='text-center py-2'>Thu</div>
        <div class='text-center py-2'>Fri</div>
        <div class='text-center py-2'>Sat</div>
        <div class='text-center py-2'>Sun</div>
        {#if $query.data?.curr1?.media}
        {#each aggregate($query.data, dayList) as { day, episodes } (day.date)}
        {@const sameMonth = isSameMonth(now, day.date)}
        <div>
          <div class='flex flex-col text-xs py-3 h-24 lg:h-48' class:opacity-30={!sameMonth}>
            {#if !$breakpoints.lg}
              <Drawer.Root shouldScaleBackground portal='html'>
                <Drawer.Trigger class='h-full flex flex-col'>
                  <div class={cn('w-6 h-6 flex items-center justify-center font-bold mx-3', isToday(day.date) && 'bg-[rgb(61,180,242)] rounded-full')}>
                    {day.number}
                  </div>
                  {#if episodes.length}
                    <div class='px-3 mt-auto text-ellipsis overflow-hidden text-nowrap w-full'>
                      {episodes.length} ep{episodes.length > 1 ? 's' : ''}
                    </div>
                  {/if}
                </Drawer.Trigger>
                <Drawer.Content tabindex={null}>
                  <Drawer.Header>
                    <Drawer.Close class='ring-offset-background focus:ring-ring data-[state=open]:bg-accent data-[state=open]:text-muted-foreground absolute right-4 top-4 rounded-sm opacity-70 transition-opacity select:opacity-100 focus:outline-none focus:ring-2 focus:ring-offset-2 disabled:pointer-events-none'>
                      <Cross2 class='h-4 w-4' />
                      <span class='sr-only'>Close</span>
                    </Drawer.Close>
                  </Drawer.Header>
                  <Drawer.Footer>
                    {#each episodes as episode, i (i)}
                      {@const status = _list(episode)}
                      <ButtonPrimitive.Root class={cn('flex items-center h-4 w-full group mt-1.5 px-3', +episode.airTime < Date.now() && 'opacity-30')} href='/app/anime/{episode.id}'>
                        <div class={cn('font-medium text-nowrap text-ellipsis overflow-hidden pr-2', +episode.airTime < Date.now() && 'line-through')} title={episode.title?.userPreferred}>
                          {#if status}
                            <StatusDot variant={status} class='hidden' />
                          {/if}
                          {episode.title?.userPreferred}
                        </div>
                        <div class='ml-auto mr-1 text-nowrap'>#{episode.episode}</div>
                        <div class='text-neutral-400 group-select:text-neutral-200'>{format(episode.airTime, 'HH:mm')}</div>
                      </ButtonPrimitive.Root>
                    {/each}
                  </Drawer.Footer>
                </Drawer.Content>
              </Drawer.Root>
            {:else}
              <div class={cn('w-6 h-6 flex items-center justify-center font-bold mx-3', isToday(day.date) && 'bg-[rgb(61,180,242)] rounded-full')}>
                {day.number}
              </div>
              <div class='mt-auto'>
                {#each episodes.length > 6 ? episodes.slice(0, 5) : episodes as episode, i (i)}
                  {@const status = _list(episode)}
                  <ButtonPrimitive.Root class={cn('flex items-center h-4 w-full group mt-1.5 px-3', +episode.airTime < Date.now() && 'opacity-30')} href='/app/anime/{episode.id}'>
                    <div class={cn('font-medium text-nowrap text-ellipsis overflow-hidden pr-2', +episode.airTime < Date.now() && 'line-through')} title={episode.title?.userPreferred}>
                      {#if status}
                        <StatusDot variant={status} class='hidden xl:inline-flex' />
                      {/if}
                      {episode.title?.userPreferred}
                    </div>
                    <div class='ml-auto mr-1 text-nowrap hidden xl:inline-flex'>#{episode.episode}</div>
                    <div class='text-neutral-400 group-select:text-neutral-200 ml-auto xl:ml-0'>{format(episode.airTime, 'HH:mm')}</div>
                  </ButtonPrimitive.Root>
                {/each}
                {#if episodes.length > 6}
                  <Tooltip.Root openDelay={100}>
                    <Tooltip.Trigger class='text-neutral-500 w-full text-left px-3 mt-1.5'>
                      + {episodes.length - 5} more...
                    </Tooltip.Trigger>
                    <Tooltip.Content sameWidth={true} class='text-center gap-1.5'>
                      {#each episodes.slice(5) as episode, i (i)}
                        {@const status = _list(episode)}
                        <ButtonPrimitive.Root class={cn('flex items-center h-4 w-full group', +episode.airTime < Date.now() && 'text-neutral-400')} href='/app/anime/{episode.id}'>
                          <div class={cn('font-medium text-nowrap text-ellipsis overflow-hidden pr-2', +episode.airTime < Date.now() && 'line-through')} title={episode.title?.userPreferred}>
                            {#if status}
                              <StatusDot variant={status} class='hidden xl:inline-flex' />
                            {/if}
                            {episode.title?.userPreferred}
                          </div>
                          <div class='ml-auto mr-1 text-nowrap hidden xl:inline-flex'>#{episode.episode}</div>
                          <div class='text-neutral-400 group-select:text-neutral-900 ml-auto xl:ml-0'>{format(episode.airTime, 'HH:mm')}</div>
                        </ButtonPrimitive.Root>
                      {/each}
                    </Tooltip.Content>
                  </Tooltip.Root>
                {/if}
              </div>
            {/if}
          </div>
        </div>
        {/each}
        {:else}
        {#each dayList as { date, number } (date)}
          {@const sameMonth = isSameMonth(now, date)}
          <div>
            <div class='flex flex-col text-xs py-3 h-48' class:opacity-30={!sameMonth}>
              <div class={cn('w-6 h-6 flex items-center justify-center font-bold mx-3', isToday(date) && 'bg-[rgb(61,180,242)] rounded-full')}>
                {number}
              </div>
            </div>
          </div>
        {/each}
        {/if}
      </div>
    {/if}
  {/if}
</div>
