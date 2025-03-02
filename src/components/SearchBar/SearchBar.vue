<script setup lang="ts">
import { onKeyStroke, useDebounceFn } from '@vueuse/core'
import DOMPurify from 'dompurify'

import { settings } from '~/logic'
import api from '~/utils/api'
import { findLeafActiveElement } from '~/utils/element'
import { isHomePage } from '~/utils/main'

import type { HistoryItem, SuggestionItem, SuggestionResponse } from './searchHistoryProvider'
import {
  addSearchHistory,
  clearAllSearchHistory,
  getSearchHistory,
  removeSearchHistory,
} from './searchHistoryProvider'

defineProps<{
  darkenOnFocus?: boolean
  blurredOnFocus?: boolean
  focusedCharacter?: string
}>()

const keywordRef = ref<HTMLInputElement>()
const isFocus = ref<boolean>(false)
const keyword = ref<string>('')
const suggestions = reactive<SuggestionItem[]>([])
const selectedIndex = ref<number>(-1)
const searchHistory = shallowRef<HistoryItem[]>([])
const historyItemRef = ref<HTMLElement[]>([])
const suggestionItemRef = ref<HTMLElement[]>([])

watch(isFocus, async (focus) => {
  // 延后加载搜索历史
  if (focus)
    searchHistory.value = await getSearchHistory()
})

onKeyStroke('/', (e: KeyboardEvent) => {
  // Reference: https://github.com/polywock/globalSpeed/blob/3705ac836402b324550caf92aa65075b2f2347c6/src/contentScript/ConfigSync.ts#L94
  const target = e.target as HTMLElement
  const ignoreTagNames = ['INPUT', 'TEXTAREA']
  if (target && (ignoreTagNames.includes(target.tagName) || target.isContentEditable))
    return

  const activeElement = findLeafActiveElement(document) as HTMLElement | undefined
  if (activeElement && target !== activeElement) {
    if (ignoreTagNames.includes(activeElement.tagName) || activeElement.isContentEditable)
      return
  }

  e.preventDefault()
  keywordRef.value?.focus()
})
onKeyStroke('Escape', (e: KeyboardEvent) => {
  e.preventDefault()
  keywordRef.value?.blur()
  isFocus.value = false
}, { target: keywordRef })

const handleInput = useDebounceFn(() => {
  selectedIndex.value = -1
  if (keyword.value.trim().length > 0) {
    api.search.getSearchSuggestion({
      term: keyword.value,
    })
      .then((res: SuggestionResponse) => {
        if (!res || (res && res.code !== 0))
          return
        Object.assign(suggestions, res.result.tag)
      })
  }
  else {
    suggestions.length = 0
  }
}, 200)

async function navigateToSearchResultPage(keyword: string) {
  if (keyword) {
    let target = '_blank'
    if (settings.value.searchBarLinkOpenMode === 'currentTabIfNotHomepage')
      target = isHomePage() ? '_blank' : '_self'
    else if (settings.value.searchBarLinkOpenMode === 'currentTab')
      target = '_self'
    else if (settings.value.searchBarLinkOpenMode === 'newTab')
      target = '_blank'

    window.open(`//search.bilibili.com/all?keyword=${encodeURIComponent(keyword)}`, target)
    const searchItem = {
      value: keyword,
      timestamp: Number(new Date()),
    }
    searchHistory.value = await addSearchHistory(searchItem)
  }
}

async function handleDelete(value: string) {
  searchHistory.value = await removeSearchHistory(value)
}

function handleKeyUp(e: KeyboardEvent) {
  // Skip the key event triggered by IME
  if (e.isComposing)
    return

  if (selectedIndex.value <= 0) {
    selectedIndex.value = 0
    return
  }

  selectedIndex.value--

  if (isFocus.value && suggestions.length !== 0)
    keyword.value = suggestions[selectedIndex.value].value
  else if (isFocus.value && searchHistory.value.length !== 0)
    keyword.value = searchHistory.value[selectedIndex.value].value

  suggestionItemRef.value.forEach((item, index) => {
    if (index === selectedIndex.value)
      item.classList.add('active')
    else item.classList.remove('active')
  })

  historyItemRef.value.forEach((item, index) => {
    if (index === selectedIndex.value)
      item.classList.add('active')
    else item.classList.remove('active')
  })
}

function handleKeyDown(e: KeyboardEvent) {
  // Skip the key event triggered by IME
  if (e.isComposing)
    return

  let isShowSuggestion = false
  if (isFocus.value && suggestions.length !== 0)
    isShowSuggestion = true
  else if (isFocus.value && !keyword.value && searchHistory.value.length !== 0)
    isShowSuggestion = false

  if (
    isShowSuggestion
    && selectedIndex.value >= suggestions.length - 1
  ) {
    selectedIndex.value = suggestions.length - 1
    return
  }
  if (
    !isShowSuggestion
    && selectedIndex.value >= searchHistory.value.length - 1
  ) {
    selectedIndex.value = searchHistory.value.length - 1
    return
  }

  selectedIndex.value++
  keyword.value = isShowSuggestion
    ? suggestions[selectedIndex.value].value
    : searchHistory.value[selectedIndex.value].value

  suggestionItemRef.value.forEach((item, index) => {
    if (index === selectedIndex.value)
      item.classList.add('active')
    else item.classList.remove('active')
  })

  historyItemRef.value.forEach((item, index) => {
    if (index === selectedIndex.value)
      item.classList.add('active')
    else item.classList.remove('active')
  })
}

function handleKeyEnter(e: KeyboardEvent) {
  if (!e.shiftKey && e.key === 'Enter' && !e.isComposing) {
    e.preventDefault()
    navigateToSearchResultPage(keyword.value)
  }
}

async function handleClearSearchHistory() {
  await clearAllSearchHistory()
  searchHistory.value = []
}
</script>

<template>
  <div id="search-wrap" w="full" max-w="550px" h-46px pos="relative">
    <div
      v-if="!darkenOnFocus && isFocus"
      pos="fixed top-0 left-0"
      w="full"
      h="full"
      content="~"
      @click="isFocus = false"
    />
    <Transition name="mask">
      <div
        v-if="darkenOnFocus && isFocus" pos="fixed top-0 left-0" w-full h-full bg="black opacity-60"
        @click="isFocus = false"
      />
    </Transition>

    <div
      v-if="blurredOnFocus"
      pos="fixed top-0 left-0" w-full h-full duration-500 pointer-events-none
      ease-out transform-gpu
      :style="{ backdropFilter: isFocus ? 'blur(15px)' : 'blur(0)' }"
    />

    <div
      class="search-bar group"
      :class="isFocus ? 'focus' : ''"
      flex="~ items-center" pos="relative"
      h-inherit
    >
      <Transition name="focus-character">
        <img
          v-show="focusedCharacter && isFocus" :src="focusedCharacter"
          width="100" object-contain pos="absolute right-0 bottom-40px"
        >
      </Transition>

      <input
        ref="keywordRef"
        v-model="keyword"
        class="group"
        rounded="60px focus:$bew-radius"
        p="l-6 r-18 y-3"
        h-inherit
        text="$b-search-bar-normal-text-color group-focus-within:$b-search-bar-focus-text-color group-hover:$b-search-bar-hover-text-color"
        un-border="1 solid $bew-border-color focus:$bew-theme-color"
        transition="all duration-300"
        type="text"
        @focus="isFocus = true"
        @input="handleInput"
        @keydown.enter.stop.passive="handleKeyEnter"
        @keyup.up.stop.passive="handleKeyUp"
        @keyup.down.stop.passive="handleKeyDown"
        @keydown.stop="() => {}"
      >
      <button
        v-if="isFocus && keyword"
        pos="absolute right-12" bg="$bew-fill-1 hover:$bew-fill-2" text="xs" rounded-10
        p-1
        flex="~ items-center justify-between"
        @click="keyword = ''"
      >
        <div i-ic-baseline-clear shrink-0 />
      </button>

      <button
        p-2
        rounded-full
        text="lg leading-0 $b-search-bar-normal-icon-color group-hover:$b-search-bar-hover-icon-color group-focus-within:$b-search-bar-focus-icon-color"
        transition="all duration-300"
        border-none
        outline-none
        pos="absolute right-6px"
        bg="hover:$bew-fill-2"
        filter="group-focus-within:~"
        style="--un-drop-shadow: drop-shadow(0 0 6px var(--bew-theme-color))"
        @click="navigateToSearchResultPage(keyword)"
      >
        <div i-tabler:search block align-middle />
      </button>
    </div>

    <Transition name="result-list">
      <div
        v-if="
          isFocus
            && searchHistory.length !== 0
            && keyword.length === 0
        "
        id="search-history"
      >
        <div class="history-list flex flex-col gap-y-2">
          <div class="title p-2 pb-0 flex justify-between">
            <span>{{ $t('search_bar.history_title') }}</span>
            <button class="rounded-2 duration-300 pointer-events-auto cursor-pointer" hover="text-$bew-theme-color" text="base $bew-text-2" @click="handleClearSearchHistory">
              {{ $t('search_bar.clear_history') }}
            </button>
          </div>

          <div class="history-item-container p2 flex flex-wrap gap-x-3 gap-y-3">
            <div
              v-for="item in searchHistory" :key="item.timestamp" ref="historyItemRef"
              class="history-item group"
              flex justify-between items-center
              @click="navigateToSearchResultPage(item.value)"
            >
              <span> {{ item.value }}</span>
              <button
                rounded-full duration-300 pointer-events-auto cursor-pointer p-1
                text="xs $bew-text-2 hover:white" leading-0 bg="$bew-fill-2 hover:$bew-theme-color"
                pos="absolute top-0 right-0" scale-80 opacity-0 group-hover:opacity-100
                @click.stop="handleDelete(item.value)"
              >
                <div i-ic-baseline-clear />
              </button>
            </div>
          </div>
        </div>
      </div>
    </Transition>

    <Transition name="result-list">
      <div
        v-if="isFocus && suggestions.length !== 0 && keyword.length > 0"
        id="search-suggestion"
      >
        <div
          v-for="(item, index) in suggestions"
          :key="index"
          ref="suggestionItemRef"
          class="suggestion-item"
          @click="navigateToSearchResultPage(item.value)"
        >
          <span v-html="DOMPurify.sanitize(item.name)" />
        </div>
      </div>
    </Transition>
  </div>
</template>

<style lang="scss" scoped>
::v-deep(.suggest_high_light) {
  --uno: "text-$bew-theme-color not-italic";
}

.result-list-enter-active,
.result-list-leave-active {
  --uno: "transition-all duration-300 ease-in-out";
}

.result-list-enter-from,
.result-list-leave-to {
  --uno: "transform translate-y-4 opacity-0 scale-95";
}

.focus-character-enter-active,
.focus-character-leave-active {
  --uno: "transition-all duration-300 ease-in-out";
}

.focus-character-enter-from,
.focus-character-leave-to {
  --uno: "transform translate-y-6 opacity-0";
}

.mask-enter-active,
.mask-leave-active {
  --uno: "transition-all duration-300 ease-in-out";
}

.mask-enter-from,
.mask-leave-to {
  --uno: "opacity-0";
}

.mask-enter-to,
.mask-leave-from {
  --uno: "opacity-100";
}

#search-wrap {
  --b-search-bar-normal-color: var(--bew-content);
  --b-search-bar-hover-color: var(--bew-content-hover);
  --b-search-bar-focus-color: var(--bew-content-hover);

  --b-search-bar-normal-icon-color: var(--bew-text-1);
  --b-search-bar-hover-icon-color: var(--bew-theme-color);
  --b-search-bar-focus-icon-color: var(--bew-theme-color);

  --b-search-bar-normal-text-color: var(--bew-text-1);
  --b-search-bar-hover-text-color: var(--bew-text-1);
  --b-search-bar-focus-text-color: var(--bew-text-1);

  @mixin card-content {
    --uno: "text-base outline-none w-full bg-$b-search-bar-normal-color transform-gpu border-1 border-$bew-border-color";
    --uno: "shadow-[var(--bew-shadow-2),var(--bew-shadow-edge-glow-1)]";
    backdrop-filter: var(--bew-filter-glass-1);
  }

  .search-bar {
    input {
      @include card-content;

      &:hover {
        --uno: "bg-$b-search-bar-hover-color";
      }

      &:focus {
        --uno: "bg-$b-search-bar-focus-color";
      }
    }

    &.focus input {
      --uno: "border-$bew-theme-color rounded-$bew-radius";
      box-shadow:
        0 0 0 2px var(--bew-theme-color),
        0 6px 16px var(--bew-theme-color-40),
        inset 0 0 6px var(--bew-theme-color-30);
    }
  }

  @mixin search-content {
    @include card-content;
    --uno: "p-2 mt-2 absolute rounded-$bew-radius hover:block";
  }

  @mixin search-content-item {
    --uno: "px-4 py-2 w-full rounded-$bew-radius duration-300 cursor-pointer not-first:mt-1 tracking-wider hover:bg-$bew-fill-2";
  }

  #search-history {
    @include search-content;
    --uno: "bg-$bew-elevated";

    .history-list {
      --uno: "max-h-420px important-overflow-y-auto";

      .title {
        --uno: "text-lg font-500";
      }

      .history-item-container {
        .history-item {
          --uno: "relative cursor-pointer duration-300";
          --uno: "py-2 px-6 bg-$bew-fill-1 hover:bg-$bew-theme-color-20 hover:text-$bew-theme-color rounded-$bew-radius-half";
        }
      }
    }
  }

  #search-suggestion {
    @include search-content;
    --uno: "bg-$bew-elevated";
    --uno: "max-h-420px important-overflow-y-auto";

    .suggestion-item {
      @include search-content-item;

      &.active {
        --uno: "bg-$bew-fill-2 shadow-[var(--bew-shadow-1),var(--bew-shadow-edge-glow-1)]";
      }
    }
  }
}
</style>
