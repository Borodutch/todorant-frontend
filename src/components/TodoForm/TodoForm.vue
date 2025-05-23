<template lang="pug">
.todo-form-content(@keydown='enableTagNumber', @keyup='disableTagNumber')
  v-textarea(
    :append-icon='text ? "clear" : undefined',
    @click:append='clearTodoText',
    :label='$t("todo.create.text")',
    :hint='$t("todo.create.textHint")',
    :rules='textRules',
    v-model='text',
    v-on:keyup='keyUp',
    v-on:keydown='keyDown',
    v-on:keyup.esc='escape',
    ref='textInput',
    auto-grow,
    no-resize,
    rows='1',
    :class='"todo-form__textarea " + (filteredTags.length ? "pb-2" : "pb-4")',
    :disabled='todo && todo.encrypted && (errorDecrypting(todo) || !$store.state.UserStore.password)',
    :autofocus='shouldAutofocus'
  )
  .mb-4(v-if='filteredTags.length')
    v-btn.tag-text-add-todo(
      text,
      small,
      v-for='(tag, i) in filteredTags',
      :key='i',
      :color='colorForTag(tag)',
      v-hotkey.forbidden='keymap',
      @click='tagSelected(tag)'
    ) {{ "#" }}{{ tag.tag }} {{ showTagsHotkeys && i <= 8 ? `(${i + 1})` : "" }}
  v-row(no-gutters)
    v-col(cols='12', md='6')
      v-menu(v-model='dateMenu', :close-on-content-click='false', min-width=0)
        template(v-slot:activator='{ on }')
          v-text-field.todo-form__textarea.todo-form__textarea--date.md-mr(
            clearable,
            readonly,
            :label='$t("todo.create.date")',
            v-on='on',
            v-model='todo.date',
            :rules='dateAndMonthRules'
          )
        v-date-picker(
          show-adjacent-months,
          @input='dateMenu = false',
          v-model='date',
          :min='todayFormattedForExactDate',
          :first-day-of-week='safeFirstDayOfWeek',
          :locale='locale',
          :show-current='todayFormatted'
        )
    v-col(cols='12', md='6')
      v-menu(v-model='monthMenu', :close-on-content-click='false', min-width=0)
        template(v-slot:activator='{ on }')
          v-text-field.todo-form__textarea.todo-form__textarea--date.md-ml(
            clearable,
            readonly,
            :label='$t("todo.create.month")',
            v-on='on',
            v-model='todo.monthAndYear',
            :rules='dateAndMonthRules'
          )
        v-date-picker(
          show-adjacent-months,
          @input='monthMenu = false',
          v-model='monthAndYear',
          :min='todayFormattedForDatePicker',
          type='month',
          :locale='locale'
        )
    v-col(
      v-if='showMoreByDefault || moreShown || todo.time',
      cols='12',
      md='6'
    )
      v-menu(v-model='timeMenu', :close-on-content-click='false', min-width=0)
        template(v-slot:activator='{ on }')
          v-text-field.todo-form__textarea.todo-form__textarea--time.md-mr(
            clearable,
            readonly,
            :label='$t("addTodoTime")',
            v-on='on',
            v-model='todo.time'
          )
        v-card
          v-card-text
            v-time-picker.elevation-0(
              v-model='todoTime',
              format='24hr',
              :close-on-content-click='false'
            )
          v-card-actions
            v-spacer
            v-btn(text, color='blue', @click='timeMenu = false') Close
  v-row.pt-4(no-gutters)
    v-col(cols='12', md='6')
      v-switch.todo-form__select(
        :label='$t("todo.create.frog")',
        v-model='todo.frog'
      )
    v-col(cols='12', md='6')
      v-switch.todo-form__select(
        :label='$t("completed")',
        v-model='todo.completed'
      )
    v-col(
      v-if='!editTodo && (showMoreByDefault || moreShown || todo.time)',
      cols='12',
      md='6'
    )
      v-switch.todo-form__select(
        :label='$t("todo.create.goFirst")',
        v-model='todo.goFirst'
      )
    v-col(
      v-if='showMoreByDefault || moreShown || todo.time',
      cols='12',
      md='6'
    )
      v-switch.todo-form__select(
        :label='$t("breakdownMessage.title")',
        v-model='todo.repetitive'
      )
    v-col(
      v-if='!editTodo && delegates.length && (showMoreByDefault || moreShown || todo.time)',
      cols='12',
      md='6'
    )
      v-select(
        :items='delegatesItems',
        clearable,
        v-model='todo.delegator',
        :label='$t("delegate.pickDelegateField")'
      )
  v-row.v-flex-row
    v-btn(
      v-if='!showMoreByDefault && !moreShown && !todo.time',
      icon,
      text,
      color='default',
      @click='moreShown = !moreShown'
    )
      v-icon more_horiz
    slot
</template>

<script lang="ts">
import Vue from 'vue'
import Component from 'vue-class-component'
import { Todo } from '@/models/Todo'
import { i18n } from '@/plugins/i18n'
import dayjs from 'dayjs'
import { Tag } from '@/models/Tag'
import { decrypt, encrypt } from '@/utils/encryption'
import { Prop } from 'vue-property-decorator'
import { namespace } from 'vuex-class'
import { User } from '@/models/User'

import localizedFormat from 'dayjs/plugin/localizedFormat'
import enLocale from 'dayjs/locale/en'
import { serverBus } from '@/main'

dayjs.extend(localizedFormat)

const AppStore = namespace('AppStore')
const TagsStore = namespace('TagsStore')
const SettingsStore = namespace('SettingsStore')
const DelegationStore = namespace('DelegationStore')

@Component
export default class TodoForm extends Vue {
  @Prop({ required: true }) todo!: Todo
  @Prop({ required: true }) escapePressed!: () => void
  @Prop() addTodo?: () => void
  @Prop({ default: false }) editTodo!: boolean
  @Prop({ default: true }) shouldAutofocus!: boolean

  @AppStore.State language?: string
  @AppStore.State dark!: boolean
  @AppStore.State todayDateTitle!: string
  @TagsStore.State tags!: Tag[]
  @SettingsStore.State showMoreByDefault!: boolean
  @SettingsStore.State newLineOnReturn!: boolean
  @SettingsStore.State firstDayOfWeek?: number
  @SettingsStore.State hotKeysEnabled!: boolean
  @DelegationStore.State delegates!: User[]

  dateMenu = false
  monthMenu = false
  timeMenu = false
  moreShown = false
  showTagsHotkeys = false

  get keymap() {
    const tagsHotkeys = {} as { [key: string]: Function }
    this.filteredTags.forEach((_, index) => {
      tagsHotkeys[`ctrl+${index + 1}`] = () => {
        this.tagSelected(this.filteredTags[index], true)
      }
    })
    return tagsHotkeys
  }

  created() {
    dayjs.locale(enLocale)
  }

  mounted() {
    if (this.shouldAutofocus) {
      // A hack for the desktop macOS version
      setTimeout(() => {
        ;(this.$refs.textInput as any).focus()
      }, 500)
    }
  }

  get locale() {
    return this.language === 'ua' ? 'uk' : this.language
  }

  get todoTime() {
    return this.todo.time
  }
  set todoTime(newValue: any) {
    this.todo.time = newValue
    if (!this.todo.date && !this.todo.monthAndYear) {
      this.todo.date = new Date()
        .toISOString()
        .substr(0, 10)
    }
  }

  get text() {
    if (this.todo.encrypted) {
      return (
        decrypt(this.todo.text, true) ||
        (i18n.t('encryption.errorDecrypting') as string)
      )
    } else {
      const extensionText = this.$router.currentRoute.query.extension as string
      if (extensionText) {
        this.todo.text = extensionText
        this.$router.push({ path: 'superpower' })
      }
      return this.todo.text
    }
  }
  set text(newText: string) {
    if (this.todo.encrypted) {
      this.todo.text = encrypt(newText)
    } else {
      this.todo.text = newText
    }
  }

  get date() {
    return this.todo.date
  }
  set date(newDate: any) {
    // Have to use this hack here because in this case we want monthAndYear to be empty
    ;(this.todo as any).monthAndYear = undefined
    this.todo.date = newDate
  }

  get monthAndYear() {
    return this.todo.monthAndYear
  }
  set monthAndYear(newMonthAndYear: any) {
    // Have to use this hack here because in this case we want date to be empty
    ;(this.todo as any).date = undefined
    this.todo.monthAndYear = newMonthAndYear
  }

  get delegatesItems() {
    return this.delegates.map((d) => ({
      value: d._id,
      text: d.name,
    }))
  }

  errorDecrypting(todo: Todo) {
    if (todo.encrypted) {
      return !decrypt(todo.text, true)
    } else {
      return false
    }
  }

  get filteredTags() {
    let text = this.todo.text.toLowerCase()
    const emptyMatches = text.match(/#$/g) || []
    if (emptyMatches.length) {
      return this.tags
    }
    const matches = text.match(/#[\u0400-\u04FFa-zA-Z_0-9]+$/g) || []
    const match = matches[0]
    if (!matches.length || !match) {
      return this.showMoreByDefault || this.moreShown ? this.tags : []
    }
    return this.tags
      .filter((tag) => tag.tag.includes(match.substr(1)))
      .filter((tag) => tag.tag !== match.substr(1))
  }

  textRules = [
    (v: any) => !!(v || '').trim() || i18n.t('errors.todo.textLenght'),
    (v: any) => v.length <= 1500 || i18n.t('errors.todo.more1500'),
  ]

  dateAndMonthRules = [
    (v: any) => {
      const todo = (this as any).todo as Partial<Todo>
      return (
        !!todo.date ||
        !!todo.monthAndYear ||
        !!todo.delegator ||
        i18n.t('errors.todo.dateOrMonth')
      )
    },
  ]

  get safeFirstDayOfWeek() {
    const storeFirstDayOfWeek = this.firstDayOfWeek
    return storeFirstDayOfWeek === undefined
      ? this.language === 'en'
        ? 0
        : 1
      : storeFirstDayOfWeek
  }

  get todayFormatted() {
    return dayjs(this.todayDateTitle).format('YYYY-MM-DD')
  }

  get todayFormattedForExactDate() {
    return dayjs(this.todayDateTitle).format()
  }

  get todayFormattedForDatePicker() {
    const date = new Date(this.todayDateTitle)
    date.setMonth(date.getMonth() + 1)
    return dayjs(date).format()
  }

  clearTodoText() {
    this.todo.text = ''
  }

  enterPressed = true
  shiftPressed = true

  shiftUpBeforeEnter = true

  keyUp(e: KeyboardEvent) {
    if (this.newLineOnReturn) {
      if (e.key === 'Enter' && e.shiftKey) {
        e.preventDefault()
      }
    } else {
      if (e.key === 'Enter' && !e.shiftKey) {
        e.preventDefault()
      }
    }
    if (e.type === 'keydown') {
      if (e.key === 'Enter') {
        this.enterPressed = true
      }
      if (e.key === 'Shift') {
        this.shiftPressed = true
      }
    }
    if (e.type === 'keyup') {
      if (e.key === 'Enter') {
        if (this.shiftPressed) {
          this.shiftUpBeforeEnter = true
        }
        this.enterPressed = false
      }
      if (e.key === 'Shift') {
        this.shiftPressed = false
        if (this.enterPressed) {
          this.shiftUpBeforeEnter = true
          serverBus.$emit('shiftBeforeEnter')
        }
      }
    }
  }

  keyDown(e: KeyboardEvent) {
    if (this.newLineOnReturn) {
      if (e.key === 'Enter' && e.shiftKey) {
        e.preventDefault()
      }
    } else {
      if (e.key === 'Enter' && !e.shiftKey) {
        e.preventDefault()
      }
    }
    if (e.type === 'keydown') {
      if (e.key === 'Enter') {
        this.enterPressed = true
      }
      if (e.key === 'Shift') {
        this.shiftPressed = true
      }
    }
    if (e.type === 'keyup') {
      if (e.key === 'Enter') {
        if (this.shiftPressed) {
          this.shiftUpBeforeEnter = true
        }
        this.enterPressed = false
      }
      if (e.key === 'Shift') {
        this.shiftPressed = false
        if (this.enterPressed) {
          this.shiftUpBeforeEnter = true
        }
      }
    }
  }

  enableTagNumber({ code }: KeyboardEvent) {
    if (code !== 'ControlLeft' || !this.hotKeysEnabled) return
    this.showTagsHotkeys = true
  }

  disableTagNumber({ code }: KeyboardEvent) {
    if (code !== 'ControlLeft') return
    this.showTagsHotkeys = false
  }

  escape() {
    ;(this as any).escapePressed()
  }

  colorForTag(tag: Tag) {
    return tag.color || (this.dark ? '#64B5F6' : '#1E88E5')
  }

  tagSelected(tag: Tag, hotkey = false) {
    if (hotkey && !this.hotKeysEnabled) return
    const textInput = (this.$refs.textInput as any).$refs.input
    const text = this.todo.text
    const len = text.length
    let pos = textInput.selectionStart
    if (pos === undefined) {
      pos = 0
    }
    const before = text.substr(0, pos)
    const after = text.substr(pos, len)
    const insertText =
      before.length && before[before.length - 1] !== ' '
        ? ` #${tag.tag} `
        : `#${tag.tag} `
    const endPos = pos + insertText.length
    const bodyTextInput = (this.$refs.textInput as any).$el.querySelector(
      'textarea'
    )

    const emptyMatches = this.todo.text.match(/#$/g) || []
    if (emptyMatches.length) {
      this.todo.text = `${before}${tag.tag}${after}`
      ;(this.$refs.textInput as any).focus()
      setTimeout(() => bodyTextInput.setSelectionRange(endPos, endPos))
      return
    }
    const matches =
      this.todo.text.match(/#[\u0400-\u04FFa-zA-Z_0-9]+(?!\s)$/g) || []
    const match = matches[0]
    if (!matches.length || !match) {
      this.todo.text = `${before}${insertText}${after}`
      ;(this.$refs.textInput as any).focus()
      setTimeout(() => bodyTextInput.setSelectionRange(endPos, endPos))
      return
    }
    this.todo.text = `${before.substr(
      0,
      before.length - match.length
    )}${insertText}${after}`
    ;(this.$refs.textInput as any).focus()
    setTimeout(() => bodyTextInput.setSelectionRange(endPos, endPos))
  }
}
</script>

<style lang="scss">
@import 'TodoForm';
.v-input {
  padding: 0;
  margin: 0;
}

.tag-text-add-todo {
  max-width: 100%;
}

.tag-text-add-todo * {
  display: inline-block;
  text-overflow: ellipsis;
  overflow: hidden;
  width: 100%;
}
</style>
