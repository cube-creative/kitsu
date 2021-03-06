<template>
<div class="page">
  <div class="page-header">
    <div class="flexrow">
      <div class="flexrow-item">
        <entity-thumbnail
          class="asset-thumbnail"
          :entity="currentAsset"
          :empty-width="100"
          :empty-height="66.6"
          v-if="currentAsset"
        >
        </entity-thumbnail>
      </div>
      <div class="flexrow-item">
        <page-title :text="title" class="entity-title"></page-title>
      </div>
    </div>
  </div>

  <div class="columns">
    <div class="column task-column">
      <page-subtitle :text="$t('assets.tasks')"></page-subtitle>
      <entity-task-list
        :entries="currentAsset ? currentAsset.tasks : []"
        :is-loading="!currentAsset"
        :is-error="false"
      ></entity-task-list>
    </div>
    <div class="column">
      <page-subtitle :text="$t('main.info')"></page-subtitle>
      <table class="table" v-if="currentAsset">
        <tbody>
          <tr>
            <td class="field-label">{{ $t('assets.fields.description') }}</td>
            <description-cell
              :entry="currentAsset"
              :full="true"
            >
            </description-cell>
          </tr>
        </tbody>
      </table>
    </div>
  </div>

  <div class="asset-casted-in">
    <page-subtitle :text="$t('assets.cast_in')"></page-subtitle>
    <div v-if="currentAsset">
      <div
        class="sequence-shots"
        :key="sequenceShots.length > 0 ? sequenceShots[0].sequence_name : ''"
        v-for="sequenceShots in currentAsset.castInShotsBySequence"
        v-if="currentAsset.castInShotsBySequence[0].length > 0"
      >
        <div class="shot-sequence">
          {{ sequenceShots.length > 0 ? sequenceShots[0].sequence_name : '' }}
        </div>
        <div class="shot-list">
          <router-link
            class="shot-link"
            :key="shot.shot_id"
            :to="{
              name: 'shot',
              params: {
                production_id: currentProduction.id,
                shot_id: shot.shot_id
              }
            }"
            v-for="shot in sequenceShots"
          >
            <entity-thumbnail
              :entity="shot"
              :square="true"
              :empty-width="100"
              :empty-height="100"
              :with-link="false"
            >
            </entity-thumbnail>
            <div>
              <span>{{ shot.name }}</span>
              <span v-if="shot.nb_occurences > 1">
                ({{ shot.nb_occurences }})
              </span>
            </div>
          </router-link>
        </div>
      </div>
      <div v-else>
        {{ $t('assets.no_cast_in') }}
      </div>
    </div>
    <table-info
      :is-loading="castIn.isLoadin"
      :is-error="castIn.isError"
      v-else
    >
    </table-info>
  </div>
</div>
</template>

<script>
import { mapGetters, mapActions } from 'vuex'

import DescriptionCell from './cells/DescriptionCell'
import PageTitle from './widgets/PageTitle'
import PageSubtitle from './widgets/PageSubtitle'
import EntityThumbnail from './widgets/EntityThumbnail'
import TableInfo from './widgets/TableInfo'
import EntityTaskList from './lists/EntityTaskList'

export default {
  name: 'asset',
  components: {
    DescriptionCell,
    EntityThumbnail,
    EntityTaskList,
    PageSubtitle,
    PageTitle,
    TableInfo
  },

  data () {
    return {
      currentAsset: null,
      castIn: {
        isLoading: false,
        isError: false
      }
    }
  },

  created () {
    this.clearSelectedTasks()
    this.currentAsset = this.getCurrentAsset()

    this.castIn.isLoading = true
    this.castIn.isError = false

    if (!this.currentAsset) {
      this.loadAsset({
        assetId: this.route.params.asset_id,
        callback: (err) => {
          if (!err) {
            this.currentAsset = this.getCurrentAsset()
            this.loadAssetCastIn({
              asset: this.currentAsset,
              callback: (err, castIn) => {
                if (err) {
                  this.castIn.isError = true
                } else {
                  this.castIn.isError = false
                }
                this.castIn.isLoading = true
              }
            })
          }
        }
      })
    } else {
      this.loadAssetCastIn({
        asset: this.currentAsset,
        callback: (err, castIn) => {
          if (err) {
            this.castIn.isError = true
          } else {
            this.castIn.isError = false
          }
          this.castIn.isLoading = true
        }
      })
    }
  },

  computed: {
    ...mapGetters([
      'assetMap',
      'currentProduction',
      'route'
    ]),

    title () {
      if (this.currentAsset) {
        return `${this.currentAsset.project_name} / ` +
               `${this.currentAsset.asset_type_name} / ` +
               `${this.currentAsset.name}`
      } else {
        return 'Loading...'
      }
    },

    assetThumbnailPath () {
      const previewId = this.currentAsset.preview_file_id
      return `/api/pictures/originals/preview-files/${previewId}.png`
    },

    isPreview () {
      return this.currentAsset &&
        this.currentAsset.preview_file_id &&
        this.currentAsset.preview_file_id.length > 0
    }
  },

  methods: {
    ...mapActions([
      'loadAsset',
      'loadAssetCastIn',
      'clearSelectedTasks'
    ]),
    changeTab (tab) {
      this.selectedTab = tab
    },
    getCurrentAsset () {
      return this.assetMap[this.route.params.asset_id] || null
    }
  },

  watch: {
    $route () { this.handleModalsDisplay() }
  },

  metaInfo () {
    return {
      title: `${this.title} - Kitsu`
    }
  }
}
</script>

<style scoped>
h2.subtitle {
  margin-top: 0;
  margin-bottom: 0.5em;
  font-weight: 300;
  font-size: 1.5em;
}

.page {
  background: #F9F9F9;
  padding: 0em;
}

.page-header {
  padding: 1em 1em 1em 1em;
  background: white;
  box-shadow: 0px 0px 6px #E0E0E0;
  margin-top: calc(50px + 2em);
  margin-bottom: 2em;
  margin-left: 1em;
  margin-right: 1em;
}

.columns {
  margin-left: 1em;
  margin-right: 1em;
}

.column {
  background: white;
  padding: 1em;
  box-shadow: 0px 0px 6px #E0E0E0;
}

.column:first-child {
  margin-right: 1em;
}

.asset-casted-in {
  margin-left: 1em;
  margin-right: 1em;
  background: white;
  padding: 1em;
  box-shadow: 0px 0px 6px #E0E0E0;
}

.asset-thumbnail {
  max-width: 100px;
}

.thumbnail-picture {
  margin-bottom: 0.5em;
}

.shot-sequence {
  text-transform: uppercase;
  font-size: 1.2em;
  color: #999;
  margin-top: 2em;
  margin-bottom: 0.4em;
}

.shot-list {
  display: flex;
  flex-wrap: wrap;
}

.shot-link {
  color: inherit;
  margin-right: 1em;
  display: flex;
  flex-direction: column;
  align-items: center;
  font-size: 0.8em;
}

.shot-link div {
  max-width: 100px;
}

.shot-link span {
  word-wrap: break-word;
}

.field-label {
  font-weight: bold;
  width: 120px;
}

@media screen and (max-width: 768px) {
  .task-column {
    margin-bottom: 1em;
  }

  .column:first-child {
    margin-right: 0;
  }

  .entity-title {
    font-size: 1.3em;
    line-height: 1.5em;
  }
}
</style>
