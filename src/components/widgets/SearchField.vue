<template>
<div class="flexrow">
  <div class="flexrow-item">
    <search-icon></search-icon>
  </div>

  <div class="flexrow-item search-field">
    <input
      ref="input"
      class="input search-input"
      type="text"
      :placeholder="placeholder"
      @input="onSearchChange"
      @keyup.enter="onSaveClicked"
      v-focus
    />
  </div>

  <div class="flexrow-item erase-search">
    <span
      class="tag"
      @click="clearSearch"
    >
      x
    </span>
  </div>

  <div class="flexrow-item save-search" v-if="canSave">
    <button class="button" @click="onSaveClicked">
      <save-icon class="icon is-small only-icon"></save-icon>
    </button>
  </div>
</div>
</template>

<script>
import { SaveIcon, SearchIcon, XIcon } from 'vue-feather-icons'

export default {
  name: 'search-field',

  props: {
    placeholder: {
      type: String,
      default: ''
    },
    canSave: {
      type: Boolean,
      default: false
    }
  },

  components: {
    SaveIcon,
    SearchIcon,
    XIcon
  },

  computed: {
  },

  methods: {
    onSearchChange () {
      this.$emit('change', this.$refs.input.value)
    },

    onSaveClicked () {
      const value = this.$refs.input.value.trim()
      if (value.length > 0) {
        if (this.canSave) this.$emit('save', value)
      }
    },

    getValue (value) {
      return this.$refs.input.value
    },

    setValue (value) {
      this.$refs.input.value = value
    },

    focus () {
      this.$refs.input.focus()
    },

    clearSearch () {
      this.setValue('')
      this.onSearchChange()
    }
  }
}
</script>
<style>
.erase-search {
  cursor: pointer;
  margin: 0;
  padding: 0;
}

.search-field {
  margin-right: 0.2em;
}

.search-input {
  width: 300px;
}

.search-input::placeholder {
  color: #CCC;
}

.save-search .button {
  margin-left: 0.5em;
  border: 0;
  color: #777;
}

.save-search .button:hover {
  color: #333;
}

@media screen and (max-width: 768px) {
  .search-input {
    width: 200px;
  }
}
</style>
