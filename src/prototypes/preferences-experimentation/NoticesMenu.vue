<script setup lang="ts">
import { computed, nextTick, onMounted, onUnmounted, ref, watch } from 'vue'
import { useRouter } from 'vue-router'
import { CdxButton, CdxIcon } from '@wikimedia/codex'
import {
  cdxIconArticle,
  cdxIconBellOutline,
  cdxIconNotice,
  cdxIconSettings,
  cdxIconTray,
} from '@wikimedia/codex-icons'

import { EXPERIMENTATION_PREFERENCES } from './routes'

const props = withDefaults(
  defineProps<{
    /** Open the Notices panel on mount (Main Page delivery). */
    defaultOpen?: boolean
  }>(),
  {
    defaultOpen: false,
  },
)

type NoticeAction = {
  label: string
  icon: typeof cdxIconArticle
  to?: { path: string; hash?: string }
  href?: string
}

type NoticeItem = {
  id: string
  header: Array<{ text: string; bold?: boolean }>
  body: string
  timestamp: string
  unread: boolean
  primary?: NoticeAction
  secondary?: NoticeAction
}

const router = useRouter()
const open = ref(false)
const root = ref<HTMLElement | null>(null)
const trigger = ref<HTMLElement | null>(null)
const panel = ref<HTMLElement | null>(null)
const panelStyle = ref<Record<string, string>>({})

const notices = ref<NoticeItem[]>([
  {
    id: 'product-testing',
    header: [
      {
        text: 'We experiment with new features and collect anonymous interaction data to improve Wikipedia for everyone.',
      },
    ],
    body: 'You can change this anytime in Preferences → Data collection.',
    timestamp: 'now',
    unread: true,
    primary: {
      label: 'Manage your data collection preferences',
      icon: cdxIconSettings,
      to: {
        path: EXPERIMENTATION_PREFERENCES,
        hash: '#mw-prefsection-experimentation',
      },
    },
    secondary: {
      label: 'Learn which tests this covers',
      icon: cdxIconArticle,
      href: 'https://meta.wikimedia.org/wiki/List_of_experiments_in_Product_and_Technology',
    },
  },
  {
    id: 'flow-archive-1',
    header: [
      { text: 'Topic ' },
      { text: "'Invitation to feedback session'", bold: true },
      { text: ' was archived on ' },
      { text: 'Talk:Main Page', bold: true },
      { text: '.' },
    ],
    body: 'You might no longer receive notifications about this topic.',
    timestamp: '4mo',
    unread: false,
    primary: {
      label: 'View page',
      icon: cdxIconArticle,
      href: 'https://en.wikipedia.org/wiki/Talk:Main_Page',
    },
    secondary: {
      label: 'Stop receiving notifications like this',
      icon: cdxIconBellOutline,
    },
  },
  {
    id: 'flow-archive-2',
    header: [
      { text: 'Topic ' },
      { text: "'Request for feedback on content'", bold: true },
      { text: ' was archived on ' },
      { text: 'Talk:Main Page', bold: true },
      { text: '.' },
    ],
    body: 'You might no longer receive notifications about this topic.',
    timestamp: '1yr',
    unread: false,
    primary: {
      label: 'View page',
      icon: cdxIconArticle,
      href: 'https://en.wikipedia.org/wiki/Talk:Main_Page',
    },
    secondary: {
      label: 'Stop receiving notifications like this',
      icon: cdxIconBellOutline,
    },
  },
])

const unreadCount = computed(() => notices.value.filter((n) => n.unread).length)

const badgeText = computed(() => {
  const n = unreadCount.value
  if (n <= 0) return ''
  return n > 99 ? '99+' : String(n)
})

function updatePosition() {
  const el = trigger.value
  if (!el) return
  const rect = el.getBoundingClientRect()
  const gap = 8
  panelStyle.value = {
    position: 'fixed',
    top: `${Math.round(rect.bottom + gap)}px`,
    right: `${Math.round(window.innerWidth - rect.right - 6)}px`,
    left: 'auto',
  }
}

function toggle() {
  open.value = !open.value
}

function close() {
  open.value = false
}

function markRead(id: string) {
  const item = notices.value.find((n) => n.id === id)
  if (item) item.unread = false
}

function toggleRead(id: string) {
  const item = notices.value.find((n) => n.id === id)
  if (item) item.unread = !item.unread
}

function onPrimaryClick(item: NoticeItem, event?: Event) {
  markRead(item.id)
  if (item.primary?.to) {
    event?.preventDefault()
    close()
    void router.push(item.primary.to)
  }
}

function onDocumentPointerDown(event: PointerEvent) {
  if (!open.value) return
  const target = event.target as Node | null
  if (!target) return
  if (root.value?.contains(target) || panel.value?.contains(target)) return
  close()
}

function onKeydown(event: KeyboardEvent) {
  if (event.key === 'Escape' && open.value) close()
}

function onReposition() {
  if (open.value) updatePosition()
}

watch(open, async (isOpen) => {
  if (isOpen) {
    await nextTick()
    updatePosition()
  }
})

onMounted(() => {
  document.addEventListener('pointerdown', onDocumentPointerDown)
  document.addEventListener('keydown', onKeydown)
  window.addEventListener('resize', onReposition)
  window.addEventListener('scroll', onReposition, true)
  if (props.defaultOpen) {
    open.value = true
  }
})

onUnmounted(() => {
  document.removeEventListener('pointerdown', onDocumentPointerDown)
  document.removeEventListener('keydown', onKeydown)
  window.removeEventListener('resize', onReposition)
  window.removeEventListener('scroll', onReposition, true)
})

watch(
  () => props.defaultOpen,
  (value) => {
    if (value) open.value = true
  },
)
</script>

<template>
  <div ref="root" class="notices-menu">
    <span ref="trigger" class="notices-menu__trigger">
      <CdxButton
        weight="quiet"
        class="notices-menu__button"
        :class="{ 'notices-menu__button--open': open }"
        aria-label="Notices"
        :aria-expanded="open"
        aria-haspopup="true"
        aria-controls="notices-menu-panel"
        @click="toggle"
      >
        <span
          class="notices-menu__badge"
          :class="{
            'notices-menu__badge--unseen': unreadCount > 0,
            'notices-menu__badge--empty': unreadCount === 0,
          }"
          :data-counter-text="badgeText"
        >
          <CdxIcon :icon="cdxIconTray" />
        </span>
      </CdxButton>
    </span>

    <Teleport to="body">
      <div
        v-show="open"
        id="notices-menu-panel"
        ref="panel"
        class="notices-menu__panel"
        role="dialog"
        aria-label="Notices"
        :style="panelStyle"
      >
        <header class="notices-menu__head">
          <CdxIcon class="notices-menu__head-icon" :icon="cdxIconTray" />
          <span class="notices-menu__head-title">Notices</span>
        </header>

        <ul class="notices-menu__list">
          <li
            v-for="item in notices"
            :key="item.id"
            class="notices-menu__item"
            :class="{ 'notices-menu__item--unread': item.unread }"
          >
            <div class="notices-menu__icon" aria-hidden="true">
              <CdxIcon class="notices-menu__icon-glyph" :icon="cdxIconNotice" />
            </div>

            <div class="notices-menu__content">
              <button
                type="button"
                class="notices-menu__read-toggle"
                :aria-label="item.unread ? 'Mark as read' : 'Mark as unread'"
                :title="item.unread ? 'Mark as read' : 'Mark as unread'"
                @click.stop="toggleRead(item.id)"
              >
                <span
                  class="notices-menu__read-circle"
                  :class="{ 'notices-menu__read-circle--unread': item.unread }"
                />
              </button>

              <div class="notices-menu__message">
                <div class="notices-menu__header">
                  <template v-for="(part, i) in item.header" :key="i">
                    <strong v-if="part.bold">{{ part.text }}</strong>
                    <template v-else>{{ part.text }}</template>
                  </template>
                </div>
                <div class="notices-menu__body">
                  {{ item.body }}
                </div>
              </div>

              <div class="notices-menu__actions">
                <div class="notices-menu__action-buttons">
                  <a
                    v-if="item.primary?.to"
                    class="notices-menu__action"
                    :href="item.primary.to.path + (item.primary.to.hash || '')"
                    @click.stop.prevent="onPrimaryClick(item, $event)"
                  >
                    <CdxIcon size="small" :icon="item.primary.icon" />
                    <span>{{ item.primary.label }}</span>
                  </a>
                  <a
                    v-else-if="item.primary?.href"
                    class="notices-menu__action"
                    :href="item.primary.href"
                    rel="noopener noreferrer"
                    target="_blank"
                    @click="markRead(item.id)"
                  >
                    <CdxIcon size="small" :icon="item.primary.icon" />
                    <span>{{ item.primary.label }}</span>
                  </a>

                  <a
                    v-if="item.secondary?.href"
                    class="notices-menu__action"
                    :href="item.secondary.href"
                    rel="noopener noreferrer"
                    target="_blank"
                    @click="markRead(item.id)"
                  >
                    <CdxIcon size="small" :icon="item.secondary.icon" />
                    <span>{{ item.secondary.label }}</span>
                  </a>
                  <button
                    v-else-if="item.secondary"
                    type="button"
                    class="notices-menu__action"
                    @click="markRead(item.id)"
                  >
                    <span
                      class="notices-menu__mute-icon"
                      :class="{
                        'notices-menu__mute-icon--slash':
                          item.secondary.icon === cdxIconBellOutline,
                      }"
                      aria-hidden="true"
                    >
                      <CdxIcon size="small" :icon="item.secondary.icon" />
                    </span>
                    <span>{{ item.secondary.label }}</span>
                  </button>
                </div>
                <span class="notices-menu__timestamp">{{ item.timestamp }}</span>
              </div>
            </div>
          </li>
        </ul>
      </div>
    </Teleport>
  </div>
</template>

<style scoped>
.notices-menu {
  display: inline-flex;
  position: relative;
}

.notices-menu__trigger {
  display: inline-flex;
}

.notices-menu__button--open {
  outline: 1px solid var(--border-color-progressive, #36c);
  outline-offset: -1px;
  background-color: var(--background-color-interactive-subtle, #eaecf0);
}

.notices-menu__badge {
  position: relative;
  display: inline-flex;
  line-height: 0;
}

.notices-menu__badge--unseen::after {
  position: absolute;
  top: -0.15em;
  left: 55%;
  min-width: 1.1em;
  padding: 0 0.3em;
  border: 1px solid var(--background-color-base, #fff);
  border-radius: 0;
  background-color: #0645ad;
  color: var(--color-inverted, #fff);
  content: attr(data-counter-text);
  font-size: 0.65rem;
  font-weight: 700;
  line-height: 1.35;
  text-align: center;
  text-indent: 0;
}

.notices-menu__badge--empty::after {
  display: none;
}

.notices-menu__panel {
  z-index: 1000;
  width: min(450px, calc(100vw - 2rem));
  background-color: var(--background-color-base, #fff);
  border: 1px solid var(--border-color-base, #a2a9b1);
  border-radius: 2px;
  box-shadow: 0 2px 2px 0 rgba(0, 0, 0, 0.2);
  font-size: 0.875rem;
}

/* OOUI-style beak under the tray icon */
.notices-menu__panel::before,
.notices-menu__panel::after {
  content: '';
  position: absolute;
  right: 14px;
  width: 0;
  height: 0;
  border-style: solid;
  border-color: transparent;
  pointer-events: none;
}

.notices-menu__panel::before {
  top: -9px;
  border-width: 0 9px 9px;
  border-bottom-color: var(--border-color-base, #a2a9b1);
}

.notices-menu__panel::after {
  top: -8px;
  right: 15px;
  border-width: 0 8px 8px;
  border-bottom-color: var(--background-color-base, #fff);
}

.notices-menu__head {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.65rem 1rem;
  border-bottom: 1px solid var(--border-color-muted, #c8ccd1);
  font-weight: 700;
  color: var(--color-base, #202122);
}

.notices-menu__head-icon {
  color: var(--color-base, #202122);
}

.notices-menu__list {
  margin: 0;
  padding: 0;
  list-style: none;
  max-height: min(70vh, 28rem);
  overflow: auto;
}

.notices-menu__item {
  display: block;
  position: relative;
  padding: 0.8em 1em 0.5em;
  background-color: #eaecf0;
  box-sizing: border-box;
}

.notices-menu__item:not(:last-child) {
  border-bottom: 1px solid #ccc;
}

.notices-menu__item:hover {
  background-color: #ececec;
}

.notices-menu__item--unread {
  background-color: #fff;
}

.notices-menu__item--unread:hover {
  background-color: #f9f9f9;
}

.notices-menu__icon {
  position: absolute;
  top: 0.8em;
  left: 1em;
  display: flex;
  align-items: center;
  justify-content: center;
  width: 30px;
  height: 30px;
  border-radius: 50%;
  background-color: #000;
  color: #fff;
}

.notices-menu__icon-glyph {
  color: #fff;
}

.notices-menu__icon-glyph :deep(svg) {
  fill: currentColor;
}

.notices-menu__content {
  display: block;
  box-sizing: border-box;
  margin-left: 30px;
  padding: 0 0.8em 0.5em 0.8em;
}

.notices-menu__read-toggle {
  float: right;
  margin: -0.55em -0.65em 0 0;
  padding: 0;
  border: 0;
  background: transparent;
  cursor: pointer;
}

.notices-menu__read-circle {
  display: block;
  box-sizing: border-box;
  width: 0.7em;
  height: 0.7em;
  min-width: 10px;
  min-height: 10px;
  margin: 0.7em;
  border-radius: 50%;
  background-color: #0645ad;
  border: 0;
}

.notices-menu__read-circle--unread {
  background-color: #eaecf0;
  border: 1px solid #767676;
}

.notices-menu__read-toggle:hover .notices-menu__read-circle {
  background-color: #447ff5;
}

.notices-menu__read-toggle:hover .notices-menu__read-circle--unread {
  background-color: #c8ccd1;
}

.notices-menu__message {
  line-height: 1.3em;
  padding-right: 1em;
  word-break: break-word;
}

.notices-menu__header {
  color: #000;
}

.notices-menu__body {
  margin-top: 4px;
  color: #767676;
}

.notices-menu__actions {
  display: flex;
  align-items: flex-end;
  gap: 0.5rem;
  margin-top: 0.8em;
  font-size: 0.9em;
}

.notices-menu__action-buttons {
  display: flex;
  flex-wrap: wrap;
  gap: 0.35rem 1.2em;
  min-width: 0;
}

.notices-menu__action {
  display: inline-flex;
  align-items: center;
  gap: 0.35em;
  padding: 0;
  border: 0;
  background: transparent;
  color: var(--color-progressive, #36c);
  font: inherit;
  text-decoration: none;
  cursor: pointer;
  text-align: start;
}

.notices-menu__action:hover {
  text-decoration: underline;
}

.notices-menu__mute-icon {
  position: relative;
  display: inline-flex;
  line-height: 0;
}

.notices-menu__mute-icon--slash::after {
  content: '';
  position: absolute;
  left: 50%;
  top: 10%;
  width: 1.5px;
  height: 80%;
  background: currentColor;
  transform: translateX(-50%) rotate(-45deg);
  transform-origin: center;
}

.notices-menu__timestamp {
  flex-grow: 1;
  text-align: right;
  color: #000;
  opacity: 0.6;
  white-space: nowrap;
}
</style>
