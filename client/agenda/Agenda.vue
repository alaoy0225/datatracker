<template lang="pug">
.agenda(
  v-if='agendaStore.isLoaded'
  :class='{ "bolder-text": agendaStore.bolderText }'
  )
  h1
    span #[strong IETF {{agendaStore.meeting.number}}] Meeting Agenda {{titleExtra}}
    .meeting-h1-badges.d-none.d-sm-flex
      span.meeting-warning(v-if='agendaStore.meeting.warningNote') {{agendaStore.meeting.warningNote}}
  h4
    span {{agendaStore.meeting.city}}, {{ meetingDate }}
    h6.float-end.d-none.d-lg-inline(v-if='meetingUpdated') #[span.text-muted Updated:] {{ meetingUpdated }}

  .agenda-topnav.my-3
    meeting-navigation
    n-button.d-none.d-sm-flex(
      quaternary
      @click='toggleSettings'
      )
      template(#icon)
        i.bi.bi-gear
      span Settings

  .row
    .col

      // ----------------------------
      // -> Subtitle + Timezone Bar
      // ----------------------------
      .row
        .col.d-none.d-sm-flex.align-items-center
          h2 {{ agendaStore.pickerMode ? 'Session Selection' : 'Schedule'}}

          n-popover(v-if='!agendaStore.infoNoteShown')
            template(#trigger)
              n-button.ms-2(text, @click='toggleInfoNote')
                i.bi.bi-info-circle.text-muted
            span Show Info Note
        .col-12.col-sm-auto.d-flex.align-items-center
          i.bi.bi-globe.me-2
          small.me-2.d-none.d-md-inline: strong Timezone:
          n-button-group.agenda-tz-selector
            n-button(
              :type='agendaStore.isTimezoneMeeting ? `primary` : `default`'
              @click='setTimezone(`meeting`)'
              ) Meeting
            n-button(
              :type='agendaStore.isTimezoneLocal ? `primary` : `default`'
              @click='setTimezone(`local`)'
              ) Local
            n-button(
              :type='agendaStore.timezone === `UTC` ? `primary` : `default`'
              @click='setTimezone(`UTC`)'
              ) UTC
          n-select.agenda-timezone-ddn(
            v-if='siteStore.viewport > 1250'
            v-model:value='agendaStore.timezone'
            :options='timezones'
            placeholder='Select Time Zone'
            filterable
            @update:value='() => { agendaStore.persistMeetingPreferences() }'
            )

      .agenda-currentwarn.alert.alert-warning.mt-3(v-if='agendaStore.isCurrentMeeting') #[strong Note:] IETF agendas are subject to change, up to and during a meeting.
      .agenda-infonote.mt-3(v-if='agendaStore.meeting.infoNote && agendaStore.infoNoteShown')
        n-popover
          template(#trigger)
            n-button(
              text
              aria-label='Close Info Note'
              @click='toggleInfoNote'
              )
              i.bi.bi-x-square
          span Hide Info Note
        div(v-html='agendaStore.meeting.infoNote')

      // -----------------------------------
      // -> Color Legend
      // -----------------------------------

      .agenda-colorlegend.mt-3(v-if='colorLegendShown')
        div
          i.bi.bi-palette.me-2
          span Color Legend
        template(v-for='(cl, idx) of agendaStore.colors')
          div(
            v-if='cl.tag !== ``'
            :key='`cl` + idx'
            :style='{ color: cl.hex }'
            )
            span {{cl.tag}}


      // -----------------------------------
      // -> Search Bar
      // -----------------------------------

      .agenda-search.mt-3(v-if='agendaStore.searchVisible')
        n-input-group
          n-input(
            v-model:value='state.searchText'
            ref='searchIpt'
            type='text'
            placeholder='Search...'
            @keyup.esc='closeSearch'
            )
            template(#prefix)
              i.bi.bi-search.me-1
          n-popover
            template(#trigger)
              n-button(
                type='primary'
                ghost
                @click='state.searchText = ``'
                aria-label='Clear Search'
                )
                i.bi.bi-x-lg
            span Clear Search
          

      // -----------------------------------
      // -> Drawers
      // -----------------------------------
      agenda-filter
      agenda-schedule-calendar
      agenda-settings

      // -----------------------------------
      // -> Schedule List
      // -----------------------------------
      agenda-schedule-list.mt-3

    // -----------------------------------
    // -> Anchored Day Quick Access Menu
    // -----------------------------------
    .col-auto.d-print-none(v-if='siteStore.viewport >= 990')
      agenda-quick-access

  agenda-mobile-bar
</template>

<script setup>
import { computed, nextTick, onBeforeUnmount, onMounted, reactive, ref, watch } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { DateTime } from 'luxon'
import debounce from 'lodash/debounce'

import {
  NButtonGroup,
  NButton,
  NInputGroup,
  NInput,
  NPopover,
  NSelect,
  useMessage
} from 'naive-ui'
import AgendaFilter from './AgendaFilter.vue'
import AgendaScheduleList from './AgendaScheduleList.vue'
import AgendaScheduleCalendar from './AgendaScheduleCalendar.vue'
import AgendaQuickAccess from './AgendaQuickAccess.vue'
import AgendaSettings from './AgendaSettings.vue'
import AgendaMobileBar from './AgendaMobileBar.vue'
import MeetingNavigation from './MeetingNavigation.vue'

import timezones from '../shared/timezones'

import { useAgendaStore } from './store'
import { useSiteStore } from '../shared/store'

import './agenda.scss'

// MESSAGE PROVIDER

const message = useMessage()

// STORES

const agendaStore = useAgendaStore()
const siteStore = useSiteStore()

// ROUTER

const router = useRouter()
const route = useRoute()

// DATA

const state = reactive({
  searchText: '',
})

// REFS

const searchIpt = ref(null)

// WATCHERS

watch(() => agendaStore.searchVisible, (newValue) => {
  state.searchText = agendaStore.searchText
  if (newValue) {
    nextTick(() => {
      searchIpt.value?.focus()
    })
  }
})

watch(() => state.searchText, debounce((newValue) => {
  agendaStore.$patch({
    searchText: newValue.toLowerCase()
  })
}, 500))

watch(() => agendaStore.meetingDays, () => {
  nextTick(() => {
    setTimeout(() => {
      reconnectScrollObservers()
    }, 100)
  })
})

watch(() => agendaStore.isLoaded, () => {
  if (route.query.show) {
    // Handle legacy ?show= parameter
    const keywords = route.query.show.split(',').map(k => k.trim()).filter(k => !!k)
    if (keywords?.length > 0) {
      const pickedIds = []
      for (const ev of agendaStore.scheduleAdjusted) {
        if (keywords.includes(ev.sessionKeyword)) {
          pickedIds.push(ev.id)
        }
      }
      if (pickedIds.length > 0) {
        agendaStore.$patch({
          pickerMode: true,
          pickerModeView: true,
          pickedEvents: pickedIds
        })
        agendaStore.persistMeetingPreferences()
      }
    }
  }
  if (route.query.pick) {
    // Handle legacy /personalize path (open picker mode)
    agendaStore.$patch({ pickerMode: true })
    router.replace({ query: null })
  }

  handleCurrentMeetingRedirect()
})

// COMPUTED

const titleExtra = computed(() => {
  let title = ''
  if (agendaStore.timezone === 'UTC') {
    title = `${title} (UTC)`
  }
  return title
})
const meetingDate = computed(() => {
  const start = DateTime.fromISO(agendaStore.meeting.startDate, { zone: agendaStore.meeting.timezone }).setZone(agendaStore.timezone)
  const end = DateTime.fromISO(agendaStore.meeting.endDate, { zone: agendaStore.meeting.timezone }).setZone(agendaStore.timezone)
  if (start.month === end.month) {
    return `${start.toFormat('MMMM d')} - ${end.toFormat('d, y')}`
  } else {
    return `${start.toFormat('MMMM d')} - ${end.toFormat('MMMM d, y')}`
  }
})
const meetingUpdated = computed(() => {
  return agendaStore.meeting.updated ? DateTime.fromISO(agendaStore.meeting.updated).setZone(agendaStore.timezone).toFormat(`DD 'at' tt ZZZZ`) : false
})
const colorLegendShown = computed(() => {
  return agendaStore.colorPickerVisible || (agendaStore.colorLegendShown && Object.keys(agendaStore.colorAssignments).length > 0)
})

// METHODS

function switchTab (key) {
  state.currentTab = key
  window.history.pushState({}, '', key)
}

function setTimezone (tz) {
  switch (tz) {
    case 'meeting':
      agendaStore.$patch({ timezone: agendaStore.meeting.timezone })
      break
    case 'local':
      agendaStore.$patch({ timezone: DateTime.local().zoneName })
      break
    default:
      agendaStore.$patch({ timezone: tz })
      break
  }
  agendaStore.persistMeetingPreferences()
}

function closeSearch () {
  agendaStore.$patch({
    searchText: '',
    searchVisible: false
  })
}

function toggleInfoNote () {
  agendaStore.$patch({ infoNoteShown: !agendaStore.infoNoteShown })
  agendaStore.persistMeetingPreferences()
}

function toggleSettings () {
  agendaStore.$patch({
    settingsShown: !agendaStore.settingsShown
  })
}

// -> Go to current meeting if not provided
function handleCurrentMeetingRedirect () {
  if (!route.params.meetingNumber && agendaStore.meeting.number) {
    router.replace({ params: { meetingNumber: agendaStore.meeting.number } })
  }
}

// --------------------------------------------------------------------
// Handle day indicator / scroll
// --------------------------------------------------------------------

const visibleDays = []
const scrollObserver = new IntersectionObserver(entries => {
  for (const entry of entries) {
    if (entry.isIntersecting) {
      if (!visibleDays.some(e => e.id === entry.target.dataset.dayId)) {
        visibleDays.push({
          id: entry.target.dataset.dayId,
          ts: entry.target.dataset.dayTs
        })
      }
    } else {
      const idxToRemove = visibleDays.findIndex(e => e.id === entry.target.dataset.dayId)
      if (idxToRemove >= 0) {
        visibleDays.splice(idxToRemove, 1)
      }
    }
  }

  let finalDayId = agendaStore.dayIntersectId
  let earliestTs = '9'
  for (const day of visibleDays) {
    if (day.ts < earliestTs) {
      finalDayId = day.id
      earliestTs = day.ts
    }
  }

  agendaStore.$patch({ dayIntersectId: finalDayId.toString() })
}, {
  root: null,
  rootMargin: '0px',
  threshold: [0.0, 1.0]
})

function reconnectScrollObservers () {
  scrollObserver.disconnect()
  visibleDays.length = 0
  for (const mDay of agendaStore.meetingDays) {
    const el = document.getElementById(`agenda-day-${mDay.slug}`)
    el.dataset.dayId = mDay.slug.toString()
    el.dataset.dayTs = mDay.ts
    scrollObserver.observe(el)
  }
}

// MOUNTED

onMounted(() => {
  reconnectScrollObservers()
})

onBeforeUnmount(() => {
  scrollObserver.disconnect()
})

// --------------------------------------------------------------------

// MOUNTED

onMounted(() => {
  agendaStore.fetch(route.params.meetingNumber)
  
  handleCurrentMeetingRedirect()

  // -> Hide Loading Screen
  if (agendaStore.isLoaded) {
    agendaStore.hideLoadingScreen()
  }
})

// CREATED

// -> Handle loading tab directly based on URL
if (window.location.pathname.indexOf('-utc') >= 0) {
  agendaStore.$patch({ timezone: 'UTC' })
} else if (window.location.pathname.indexOf('personalize') >= 0) {
  // state.currentTab = 'personalize'
}

</script>

<style lang="scss">
@import "bootstrap/scss/functions";
@import "bootstrap/scss/variables";
@import "../shared/breakpoints";

.agenda {
  min-height: 500px;
  font-weight: 460;

  &.bolder-text {
    font-weight: 520;
  }

  &-topnav {
    position: relative;

    > button {
      position: absolute;
      top: 5px;
      right: 0;

      .bi {
        transition: transform 1s ease;
      }

      &:hover {
        .bi {
          transform: rotate(180deg);
        }
      }
    }
  }

  &-tz-selector {
    margin-right: .5rem;

    @media screen and (max-width: $bs5-break-sm) {
      margin-right: 0;
      justify-content: stretch;
      flex: 1;

      > button {
        flex: 1;
      }
    }
  }

  &-timezone-ddn {
    min-width: 350px;
  }

  &-infonote {
    border: 1px solid $blue-400;
    border-radius: .25rem;
    background: linear-gradient(to top, lighten($blue-100, 2%), lighten($blue-100, 5%));
    box-shadow: inset 0 0 0 1px #FFF;
    padding: 16px 50px 16px 16px;
    font-size: .9rem;
    color: $blue-700;
    position: relative;

    > button {
      position: absolute;
      top: 15px;
      right: 15px;
      font-size: 1.2em;
      color: $blue-400;
    }
  }

  &-colorlegend {
    display: grid;
    gap: 10px;
    grid-template-columns: repeat(auto-fit, minmax(150px, 1fr));
    border-radius: 5px;
    font-size: .8rem;
    font-weight: 600;

    > div:first-child {
      background-color: $gray-600;
      background: linear-gradient(337deg, $gray-500 20%, $gray-600 70%);
      padding: 5px 15px;
      font-size: .7rem;
      text-transform: uppercase;
      color: #FFF;
      border-radius: 5px;
      display: flex;
      align-items: center;
    }

    > div:not(:first-child) {
      display: flex;
      align-items: center;
      padding: 5px 15px;
      justify-content: center;

      &::before {
        content: '';
        display: block;
        width: 18px;
        height: 18px;
        border-radius: 9px;
        border: 2px solid rgba(0,0,0,.1);
        background-color: currentColor;
        box-shadow: 0 0 10px 0 currentColor;
        margin-right: 10px;
      }

      span {
        color: currentColor;
      }
    }
  }
}

.n-dropdown-option {
  .text-red {
    color: $red-500;
  }
  .text-brown {
    color: $orange-700;
  }
  .text-blue {
    color: $blue-600;
  }
  .text-green {
    color: $green-500;
  }
  .text-purple {
    color: $purple-500;
  }
}

@keyframes spin {
  from { transform:rotate(0deg); }
  to { transform:rotate(360deg); }
}

@keyframes warningBorderFlash {
  10% { color: #FFF; }
  50% { color: $red-300; }
  90% { color: #FFF; }
}
</style>
