<template>
  <div class="list-group shadow">
    <vue-bootstrap-typeahead-list-item
      v-for="(item, id) in matchedItems" :key="id"
      :data="item.data"
      :html-text="highlight(item.text)"
      :background-variant="backgroundVariant"
      :text-variant="textVariant"
      @click.native="handleHit(item, $event)"
      @keydown="$listeners.keydown($event)"
    >
      <template v-if="$scopedSlots.suggestion" slot="suggestion" slot-scope="{ data, htmlText }">
        <slot name="suggestion" v-bind="{ data, htmlText }" />
      </template>
    </vue-bootstrap-typeahead-list-item>
  </div>
</template>

<script>
import VueBootstrapTypeaheadListItem from './VueBootstrapTypeaheadListItem.vue'

function sanitize(text) {
  return text.replace(/</g, '&lt;').replace(/>/g, '&gt;')
}

function escapeRegExp(str) {
  return str.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
}

export default {
  name: 'VueBootstrapTypeaheadList',

  components: {
    VueBootstrapTypeaheadListItem
  },

  props: {
    data: {
      type: Array,
      required: true,
      validator: d => d instanceof Array
    },
    query: {
      type: String,
      default: ''
    },
    backgroundVariant: {
      type: String
    },
    textVariant: {
      type: String
    },
    maxMatches: {
      type: Number,
      default: 10
    },
    minMatchingChars: {
      type: Number,
      default: 2
    }
  },

  computed: {
    highlight() {
      return (text) => {
        text = sanitize(text)
        if (this.query.length === 0) {
          return text
        }
        const re = new RegExp(this.escapedQuery, 'gi')

        return text.replace(re, `<strong>$&</strong>`)
      }
    },

    escapedQuery() {
      return escapeRegExp(sanitize(this.query))
    },

    matchedItems() {
      if (this.query.length === 0 || this.query.length < this.minMatchingChars) {
        return []
      }

      const re = new RegExp(this.escapedQuery, 'gi')

      // Filter, sort, and concat
      return this.data
        .filter(i => i.text.match(re) !== null)
        .sort((a, b) => {
          const aIndex = a.text.indexOf(a.text.match(re)[0])
          const bIndex = b.text.indexOf(b.text.match(re)[0])

          if (aIndex < bIndex) { return -1 }
          if (aIndex > bIndex) { return 1 }
          return 0
        }).slice(0, this.maxMatches)
    }
  },
  mounted() {
    this.$parent.$on('selectionKeyPressed', this.handleSelectionKey.bind(this))
  },

  methods: {
    handleSelectionKey(evt) {
      if (this.data.length < 1) return

      let active = this.findActiveItemIndex()
      let ch = this.$children
      switch (evt.key) {
        case 'ArrowUp':
        case 'Up':
          this.setActiveItem((active > 0) ? (active - 1) : (ch.length - 1))
          break
        case 'ArrowDown':
        case 'Down':
          this.setActiveItem((active + 1) % ch.length)
          break
        case 'Enter':
          this.$emit('hit', this.findActiveItem())
          break
      }
    },

    findActiveItem() {
      let idx = this.findActiveItemIndex()
      return (idx < 0) ? null : this.matchedItems[idx]
    },

    findActiveItemIndex() {
      let ch = this.$children
      for (let i = 0; i < ch.length; i++) {
        let item = ch[i]
        if (item.active) {
          return i
        }
      }
      return -1
    },

    setActiveItem(idx) {
      this.$children.forEach((item, i) => {
        if (i === idx) {
          item.active = true
          item.$el.scrollIntoView(false)
        }
        else {
          item.active = false
        }
      })
    },

    handleHit(item, evt) {
      this.$emit('hit', item)
      evt.preventDefault()
    }
  }
}
</script>
