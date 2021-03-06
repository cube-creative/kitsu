<template>
<div class="data-list">
  <div class="table-header-wrapper">
    <table class="table table-header" ref="headerWrapper">
      <thead>
        <tr>
          <th class="thumbnail"></th>
          <th class="episode" ref="th-episode" v-if="!isSingleEpisode">
            {{ $t('shots.fields.episode') }}
          </th>
          <th class="sequence" ref="th-sequence" >
            {{ $t('shots.fields.sequence') }}
          </th>
          <th class="name shot-name" ref="th-shot" >
            {{ $t('shots.fields.name') }}
          </th>
          <th class="framein" v-if="isFrameIn">
            {{ $t('shots.fields.frame_in') }}
          </th>
          <th class="frameout" v-if="isFrameOut">
            {{ $t('shots.fields.frame_out') }}
          </th>
          <th class="fps" v-if="isFps">{{ $t('shots.fields.fps') }}</th>
          <th class="description" v-if="!isCurrentUserClient">
            {{ $t('shots.fields.description') }}
          </th>
          <th
            class="validation-cell"
            :key="columnId"
            :style="{
              'border-left': '1px solid ' + taskTypeMap[columnId].color,
              'background': getBackground(taskTypeMap[columnId].color)
            }"
            v-for="columnId in validationColumns">
            <router-link
              :to="{
                name: 'task-type',
                params: {
                  production_id: currentProduction.id,
                  task_type_id: columnId
                }
              }"
            >
              {{ taskTypeMap[columnId].name }}
            </router-link>
          </th>

          <th class="actions">
            <button-link
              class="is-small"
              icon="plus"
              :text="$t('tasks.create_tasks')"
              :path="{
                name: 'create-shot-tasks',
                params: {
                  production_id: currentProduction.id
                }
              }"
              v-if="isCurrentUserManager"
            >
            </button-link>
          </th>
        </tr>
      </thead>
    </table>
  </div>

  <table-info
    :is-loading="isLoading"
    :is-error="isError"
  >
  </table-info>

  <div class="has-text-centered" v-if="isEmptyList && !isCurrentUserClient">
    <p class="info">
      <img src="../../assets/illustrations/empty_shot.png" />
    </p>
    <p class="info">{{ $t('shots.empty_list') }}</p>
    <button-link
      class="level-item big-button"
      :text="$t('shots.new_shots')"
      :path="{
        name: 'manage-shots',
        params: {production_id: currentProduction.id}
      }"
    />
  </div>
  <div class="has-text-centered" v-if="isEmptyList && isCurrentUserClient">
    <p class="info">
      <img src="../../assets/illustrations/empty_shot.png" />
    </p>
    <p class="info">{{ $t('shots.empty_list_client') }}</p>
  </div>

  <div
    ref="body"
    class="table-body"
    v-infinite-scroll="loadMoreShots"
    v-scroll="onBodyScroll"
  >
    <table class="table">
      <tbody ref="body-tbody">
        <tr
          :key="shot.id"
          :class="{canceled: shot.canceled}"
          v-for="(shot, i) in entries"
        >
          <td class="thumbnail">
            <entity-thumbnail :entity="shot" />
          </td>
          <td :class="{name: !shot.canceled}" v-if="!isSingleEpisode">
            {{ shot.episode_name }}
          </td>
          <td :class="{name: !shot.canceled}">
            {{ shot.sequence_name }}
          </td>
          <td :class="{'shot-name': true, 'name': !shot.canceled}">
            <router-link :to="{
              name: 'shot',
              params: {
                production_id: shot.production_id,
                shot_id: shot.id
              }
            }">
              {{ shot.name }}
            </router-link>
          </td>
          <td class="framein" v-if="isFrameIn">
            {{ shot.data && shot.data.frame_in ? shot.data.frame_in : ''}}
          </td>
          <td class="frameout" v-if="isFrameOut">
            {{ shot.data && shot.data.frame_out ? shot.data.frame_out : ''}}
          </td>
          <td class="fps" v-if="isFps">
            {{ shot.data && shot.data.fps ? shot.data.fps : ''}}
          </td>
          <description-cell
            class="description"
            :entry="shot"
            v-if="!isCurrentUserClient"
          />
          <validation-cell
            class="unselectable validation-cell"
            :key="columnId + '-' + shot.id"
            :ref="'validation-' + i + '-' + j"
            :column="taskTypeMap[columnId]"
            :entity="shot"
            :selected="shotSelectionGrid[i][j]"
            :rowX="i"
            :columnY="j"
            @select="onTaskSelected"
            @unselect="onTaskUnselected"
            v-for="(columnId, j) in validationColumns"
          />
          <row-actions v-if="isCurrentUserManager"
            :entry="shot"
            :edit-route="{
              name: 'edit-shots',
              params: {
                shot_id: shot.id,
                production_id: currentProduction.id
              }
            }"
            :restore-route="{
              name: 'restore-shots',
              params: {
                shot_id: shot.id,
                production_id: currentProduction.id
              }
            }"
            :delete-route="{
              name: 'delete-shots',
              params: {
                shot_id: shot.id,
                production_id: currentProduction.id
              }
            }"
          />
          <td class="actions" v-else></td>
        </tr>
      </tbody>
    </table>
  </div>

  <p class="has-text-centered nb-shots" v-if="!isEmptyList">
    {{ displayedShotsLength }} {{ $tc('shots.number', displayedShotsLength) }}
  </p>

</div>
</template>

<script>
import { mapGetters, mapActions } from 'vuex'
import colors from '../../lib/colors'

import DescriptionCell from '../cells/DescriptionCell'
import ValidationCell from '../cells/ValidationCell'
import RowActions from '../widgets/RowActions'
import ButtonLink from '../widgets/ButtonLink'
import ButtonHrefLink from '../widgets/ButtonHrefLink'
import PageTitle from '../widgets/PageTitle'
import TableInfo from '../widgets/TableInfo'
import EntityThumbnail from '../widgets/EntityThumbnail'

export default {
  name: 'shot-list',

  props: [
    'entries',
    'isLoading',
    'isError',
    'validationColumns'
  ],

  data () {
    return {
      busy: false,
      lastSelection: null
    }
  },

  components: {
    ButtonLink,
    ButtonHrefLink,
    DescriptionCell,
    EntityThumbnail,
    PageTitle,
    RowActions,
    TableInfo,
    ValidationCell
  },

  computed: {
    ...mapGetters([
      'currentProduction',
      'displayedShotsLength',
      'isCurrentUserManager',
      'isCurrentUserClient',
      'isFps',
      'isFrameIn',
      'isFrameOut',
      'isSingleEpisode',
      'nbSelectedTasks',
      'shotSearchText',
      'shotSelectionGrid',
      'taskTypeMap'
    ]),
    isEmptyList () {
      return this.entries &&
             this.entries.length === 0 &&
             !this.isLoading &&
             !this.isError &&
             (!this.shotSearchText || this.shotSearchText.length === 0)
    }
  },

  methods: {
    ...mapActions([
      'displayMoreShots'
    ]),

    onHeaderScroll (event, position) {
      this.$refs.tableWrapper.scrollLeft = position.scrollLeft
    },

    onTaskSelected (validationInfo) {
      if (validationInfo.isShiftKey) {
        if (this.lastSelection) {
          let startX = this.lastSelection.x
          let endX = validationInfo.x
          let startY = this.lastSelection.y
          let endY = validationInfo.y
          if (validationInfo.x < this.lastSelection.x) {
            startX = validationInfo.x
            endX = this.lastSelection.x
          }
          if (validationInfo.y < this.lastSelection.y) {
            startY = validationInfo.y
            endY = this.lastSelection.y
          }

          for (let i = startX; i <= endX; i++) {
            for (let j = startY; j <= endY; j++) {
              const ref = 'validation-' + i + '-' + j
              const validationCell = this.$refs[ref][0]
              if (!this.shotSelectionGrid[i][j]) {
                validationCell.select({ctrlKey: true, isUserClick: false})
              }
            }
          }
        }
      } else if (!validationInfo.isCtrlKey) {
        this.$store.commit('CLEAR_SELECTED_TASKS')
      }
      this.$store.commit('ADD_SELECTED_TASK', validationInfo)

      if (!validationInfo.isShiftKey && validationInfo.isUserClick) {
        this.lastSelection = {
          x: validationInfo.x,
          y: validationInfo.y
        }
      }
    },

    onTaskUnselected (validationInfo) {
      if (!validationInfo.isCtrlKey) {
        if (this.nbSelectedTasks === 1) {
          this.$store.commit('REMOVE_SELECTED_TASK', validationInfo)
        } else {
          this.$store.commit('CLEAR_SELECTED_TASKS')
          this.$store.commit('ADD_SELECTED_TASK', validationInfo)
        }
      } else {
        this.$store.commit('REMOVE_SELECTED_TASK', validationInfo)
      }
    },

    onBodyScroll (event, position) {
      this.$refs.headerWrapper.style.left = `-${position.scrollLeft}px`
      this.$emit('scroll', position.scrollTop)
    },

    loadMoreShots () {
      this.displayMoreShots()
    },

    setScrollPosition (scrollPosition) {
      this.$refs.body.scrollTop = scrollPosition
    },

    getBackground (color) {
      return colors.hexToRGBa(color, 0.08)
    },

    resizeHeaders () {
      if (this.$refs['body-tbody'].children.length > 0) {
        let sequenceWidth, shotWidth
        if (this.isSingleEpisode) {
          sequenceWidth =
            this.$refs['body-tbody'].children[0].children[1].offsetWidth
          shotWidth =
            this.$refs['body-tbody'].children[0].children[2].offsetWidth
        } else {
          sequenceWidth =
            this.$refs['body-tbody'].children[0].children[2].offsetWidth
          shotWidth =
            this.$refs['body-tbody'].children[0].children[3].offsetWidth
          const episodeWidth =
            this.$refs['body-tbody'].children[0].children[1].offsetWidth
          this.$refs['th-episode'].style = `min-width: ${episodeWidth}px`
        }

        this.$refs['th-sequence'].style = `min-width: ${sequenceWidth}px`
        this.$refs['th-shot'].style = `min-width: ${shotWidth}px`
      }
    }
  }
}
</script>

<style scoped>
.project {
  min-width: 60px;
  width: 60px;
}

.actions {
  min-width: 100px;
}

th.actions {
  padding: 0.4em;
}

.name {
  min-width: 100px;
  width: 100px;
  font-weight: bold;
}

.name a {
  color: inherit;
}

.name.shot-name {
  min-width: 110px;
  width: 110px;
}

.episode {
  min-width: 100px;
  width: 100px;
}

.sequence {
  min-width: 100px;
  width: 100px;
  font-weight: bold;
}

.framein {
  min-width: 60px;
  width: 60px;
}

.frameout {
  min-width: 60px;
  width: 60px;
}

.fps {
  min-width: 50px;
  max-width: 50px;
  width: 50px;
}

.description {
  min-width: 200px;
  max-width: 200px;
  width: 200px;
}

.validation-cell {
  min-width: 150px;
  max-width: 150px;
  width: 150px;
}

td.name {
  font-size: 1.2em;
}

td.sequence {
  font-size: 1.2em;
}

.canceled {
  text-decoration: line-through;
}

.thumbnail {
  min-width: 50px;
  max-width: 50px;
  width: 50px;
  padding: 0;
}

.thumbnail img {
  margin-top: 5px;
}

span.thumbnail-empty {
  display: block;
  width: 50px;
  height: 30px;
  background: #F3F3F3;
}

.info {
  margin-top: 2em;
}
</style>
