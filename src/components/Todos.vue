<template>
  <div class="todos page fixed-page">
    <div class="task-tabs tabs">
      <ul>
        <li
          :class="{'is-active': isTabActive('todos')}"
        >
          <router-link :to="{
            name: 'todos',
          }">
            {{ $t('tasks.current')}}
          </router-link>
        </li>
        <li
          :class="{'is-active': isTabActive('done')}"
          @click="selectTab('done')"
        >
          <router-link :to="{
            name: 'todos-tab',
            params: {tab: 'done'}
          }">
            {{ $t('tasks.done') }} ({{ displayedDoneTasks.length }})
          </router-link>
        </li>
        <li
          :class="{'is-active': isTabActive('timesheets')}"
          @click="selectTab('timesheet')"
        >
          <router-link :to="{
            name: 'todos-tab',
            params: {tab: 'timesheets'}
          }">
            {{ $t('timesheets.title') }}
          </router-link>
        </li>
      </ul>
    </div>

    <search-field
      :class="{
        'search-field': true
      }"
      ref="todos-search-field"
      @change="onSearchChange"
      @save="saveSearchQuery"
      :can-save="true"
      v-if="!isTabActive('done')"
    />

    <div
      class="query-list"
      v-if="isTabActive('todos') || isTabActive('timesheets')"
    >
      <search-query-list
        :queries="todoSearchQueries"
        @changesearch="changeSearch"
        @removesearch="removeSearchQuery"
      />
    </div>

    <todos-list
      ref="todo-list"
      :entries="displayedTodos"
      :is-loading="isTodosLoading"
      :is-error="isTodosLoadingError"
      :selection-grid="todoSelectionGrid"
      v-if="isTabActive('todos')"
      @scroll="setTodoListScrollPosition"
    />

    <todos-list
      ref="done-list"
      :entries="displayedDoneTasks"
      :is-loading="isTodosLoading"
      :is-error="isTodosLoadingError"
      :done="true"
      v-if="isTabActive('done')"
    />

    <timesheet-list
      ref="timesheet-list"
      :tasks="loggableTodos"
      :done-tasks="loggableDoneTasks"
      :is-loading="isTodosLoading"
      :is-error="isTodosLoadingError"
      :time-spent-map="timeSpentMap"
      :time-spent-total="timeSpentTotal"
      :hide-done="todosSearchText.length > 0"
      @date-changed="onDateChanged"
      @time-spent-change="onTimeSpentChange"
      v-if="isTabActive('timesheets')"
    />
  </div>
</template>

<script>
import { mapGetters, mapActions } from 'vuex'
import moment from 'moment-timezone'

import TimesheetList from './lists/TimesheetList'
import TodosList from './lists/TodosList'
import PageTitle from './widgets/PageTitle'
import SearchField from './widgets/SearchField'
import SearchQueryList from './widgets/SearchQueryList'

export default {
  name: 'todos',
  components: {
    TimesheetList,
    TodosList,
    PageTitle,
    SearchField,
    SearchQueryList
  },

  data () {
    return {
      activeTab: 'todos',
      selectedDate: moment().format('YYYY-MM-DD')
    }
  },

  mounted () {
    this.updateActiveTab()
    if (this.todosSearchText.length > 0) {
      this.$refs['todos-search-field'].setValue(this.todosSearchText)
    }
    this.loadTodos({
      date: this.selectedDate,
      callback: () => {
        if (this.$refs['todo-list']) {
          this.$refs['todo-list'].setScrollPosition(
            this.todoListScrollPosition
          )
        }
      }
    })
  },

  computed: {
    ...mapGetters([
      'user',
      'displayedTodos',
      'displayedDoneTasks',
      'todosSearchText',
      'isTodosLoading',
      'isTodosLoadingError',
      'taskTypeMap',
      'timeSpentMap',
      'timeSpentTotal',
      'todoListScrollPosition',
      'todoSelectionGrid',
      'todoSearchQueries'
    ]),

    loggableTodos () {
      return this.displayedTodos
        .filter((task) => {
          return this.taskTypeMap[task.task_type_id].allow_timelog
        })
    },

    loggableDoneTasks () {
      return this.displayedDoneTasks
        .filter((task) => {
          return this.taskTypeMap[task.task_type_id].allow_timelog
        })
    }
  },

  methods: {
    ...mapActions([
      'loadTodos',
      'removeTodoSearch',
      'saveTodoSearch',
      'setTodoListScrollPosition',
      'setTodosSearch',
      'setTimeSpent'
    ]),

    isTabActive (tab) {
      return this.activeTab === tab
    },

    selectTab (tab) {
      this.activeTab = tab
      if (this.isTabActive('todos')) {
        setTimeout(() => {
          if (this.$refs['person-tasks-search-field']) {
            this.$refs['person-tasks-search-field'].focus()
          }
        }, 100)
      }
    },

    updateActiveTab () {
      if (['done', 'timesheets'].includes(this.$route.params.tab)) {
        this.activeTab = this.$route.params.tab
      } else {
        this.activeTab = 'todos'
      }
    },

    onSearchChange (text) {
      this.setTodosSearch(text)
    },

    changeSearch (searchQuery) {
      this.$refs['todos-search-field'].setValue(searchQuery.search_query)
      this.$refs['todos-search-field'].$emit('change', searchQuery.search_query)
    },

    saveSearchQuery (searchQuery) {
      this.saveTodoSearch(searchQuery)
        .then(() => {
        })
        .catch((err) => {
          if (err) console.log('error', err)
        })
    },

    removeSearchQuery (searchQuery) {
      this.removeTodoSearch(searchQuery)
        .then(() => {
        })
        .catch((err) => {
          if (err) console.log('error', err)
        })
    },

    onDateChanged (date) {
      this.selectedDate = moment(date).format('YYYY-MM-DD')
      this.loadTodos({
        date: this.selectedDate,
        forced: true
      })
    },

    onTimeSpentChange (timeSpentInfo) {
      timeSpentInfo.personId = this.user.id
      timeSpentInfo.date = this.selectedDate
      this.setTimeSpent(timeSpentInfo)
    },

    onAssignation (eventData) {
      if (this.user.id === eventData.person_id) {
        this.$store.dispatch('loadTodos', {
          forced: true,
          date: this.selectedDate
        })
      }
    }
  },

  socket: {
    events: {
      'task:assign' (eventData) {
        this.onAssignation(eventData)
      },

      'task:unassign' (eventData) {
        this.onAssignation(eventData)
      }
    }
  },

  watch: {
    $route () {
      this.updateActiveTab()
    }
  },

  metaInfo () {
    return {
      title: `${this.$t('tasks.my_tasks')} - Kitsu`
    }
  }
}
</script>

<style scoped>
.task-tabs {
  margin-top: 1em;
  margin-bottom: 1em;
  font-size: 1.1em;
}

.data-list {
  margin-top: 0;
}

.search-field {
  margin-top: 1em;
  margin-bottom: 1em;
}
</style>
