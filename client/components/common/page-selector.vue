<template lang="pug">
  v-dialog(
    v-model='isShown'
    max-width='850px'
    overlay-color='blue darken-4'
    overlay-opacity='.7'
    )
    v-card.page-selector
      .dialog-header.is-blue
        v-icon.mr-3(color='white') mdi-page-next-outline
        .body-1(v-if='mode === `create`') {{$t('common:pageSelector.createTitle')}}
        .body-1(v-else-if='mode === `move`') {{$t('common:pageSelector.moveTitle')}}
        .body-1(v-else-if='mode === `select`') {{$t('common:pageSelector.selectTitle')}}
        v-spacer
        v-progress-circular(
          indeterminate
          color='white'
          :size='20'
          :width='2'
          v-show='searchLoading'
          )
      .d-flex
        v-flex.grey(:class='$vuetify.theme.dark ? `darken-4` : `lighten-3`')
          div(style='height:320px;')
            vue-scroll(:ops='scrollStyle')
              v-treeview(
                :key='`pageTree-` + treeViewCacheId'
                :active.sync='currentNode'
                :open.sync='openNodes'
                :items='tree'
                dense
                expand-icon='mdi-menu-down-outline'
                item-id='path'
                item-text='title'
                activatable
                hoverable
                )
                template(slot='prepend', slot-scope='{ item, open, leaf }')
                  v-icon(v-if='item.isFolder') mdi-{{ open ? 'folder-open' : 'folder' }}
                  v-icon(v-else) mdi-text-box
      v-card-actions.grey.pa-2(:class='$vuetify.theme.dark ? `darken-2` : `lighten-1`', v-if='!mustExist')
        span(style='flex: 0 0 120px;') {{'页面位置：'}}
        v-text-field(
          ref='pathIpt'
          solo
          hide-details
          prefix='/'
          v-model='currentPath'
          flat
          disabled
          style='border-radius: 4px;'
        )
      v-card-actions.grey.pa-2(:class='$vuetify.theme.dark ? `darken-2` : `lighten-1`', v-if='!mustExist')
        span(style='flex: 0 0 120px;') {{'新页面标题：'}}
        v-text-field(
          ref='pathIpt2'
          solo
          outlined
          hide-details="auto"
          placeholder="请输入新页面标题"
          v-model='currentTitle'
          flat
          clearable
          style='border-radius: 4px 4px;'
          :rules="[titleRules.required,titleRules.currentTitle]"
        )
      v-card-actions.grey.pa-2(:class='$vuetify.theme.dark ? `darken-2` : `lighten-1`', v-if='!mustExist && mode === `create` && newFlag ')
        span(style='flex: 0 0 120px;') {{'编辑器选择：'}}
        v-select(
          solo
          flat
          hide-details
          single-line
          :items='editorList'
          item-value="type"
          item-text="text"
          style='border-radius: 4px;'
          v-model='currentEditor'
        )
      v-card-chin
        v-spacer
        v-btn(text, @click='close') {{$t('common:actions.cancel')}}
        v-btn.px-4(color='primary', @click='open', :disabled='!isValidPath')
          v-icon(left) mdi-check
          span {{$t('common:actions.select')}}
</template>

<script>
import _ from 'lodash'
import gql from 'graphql-tag'

const localeSegmentRegex = /^[A-Z]{2}(-[A-Z]{2})?$/i

/* global siteLangs, siteConfig */
const titlePattern = /^[^\/\#\.\$\^\=\*\;\:\&\?\(\)\[\]\{\}\"\'\>\<\,\@\!\%\`\~\s]*$/

export default {
  props: {
    value: {
      type: Boolean,
      default: false
    },
    path: {
      type: String,
      default: null
    },
    locale: {
      type: String,
      default: 'en'
    },
    mode: {
      type: String,
      default: 'create'
    },
    openHandler: {
      type: Function,
      default: () => {}
    },
    mustExist: {
      type: Boolean,
      default: false
    },
    newFlag: {
      type: Boolean,
      default: false
    }
  },
  data() {
    return {
      treeViewCacheId: 0,
      searchLoading: false,
      currentLocale: siteConfig.lang,
      currentFolderPath: '',
      currentDirectory: '',
      currentTitle: '',
      currentEditor: 'ckeditor',
      editorList: [{text: '富文本编辑器', type: 'ckeditor'}, {text: 'markdown编辑器', type: 'markdown'}],
      currentPage: null,
      currentNode: [0],
      openNodes: [0],
      tree: [
        {
          id: 0,
          title: '/ (root)',
          isFolder: 1,
          children: []
        }
      ],
      pages: [],
      all: [],
      namespaces: siteLangs.length ? siteLangs.map(ns => ns.code) : [siteConfig.lang],
      scrollStyle: {
        vuescroll: {},
        scrollPanel: {
          initialScrollX: 0.01, // fix scrollbar not disappearing on load
          scrollingX: false,
          speed: 50
        },
        rail: {
          gutterOfEnds: '2px'
        },
        bar: {
          onlyShowBarOnScroll: false,
          background: '#999',
          hoverStyle: {
            background: '#64B5F6'
          }
        }
      },
      titleRules: {
        required: value => !!value || '标题必须输入',
        currentTitle: value => {
          return titlePattern.test(value) || '请确保它不包含特殊字符'
        }
      }
    }
  },
  computed: {
    isShown: {
      get() { return this.value },
      set(val) { this.$emit('input', val) }
    },
    currentPath () {
      return (this.currentDirectory ? this.currentDirectory + '/' : '') + (this.currentTitle || '')
    },
    // currentPages () {
    //   return _.sortBy(_.filter(this.pages, ['parent', _.head(this.currentNode) || 0]), ['title', 'path'])
    // },
    isValidPath () {
      if (!this.currentPath) {
        return false
      }
      if (!this.currentTitle) {
        return false
      }
      if (this.mustExist && !this.currentPage) {
        return false
      }
      if (!titlePattern.test(this.currentTitle)) {
        return false
      }
      const firstSection = _.head(this.currentPath.split('/'))
      if (firstSection.length <= 1) {
        return false
      } else if (localeSegmentRegex.test(firstSection)) {
        return false
      } else if (
        _.some(['login', 'logout', 'register', 'verify', 'favicons', 'fonts', 'img', 'js', 'svg'], p => {
          return p === firstSection
        })) {
        return false
      } else {
        return true
      }
    }
  },
  watch: {
    isShown (newValue, oldValue) {
      if (newValue && !oldValue) {
        this.fetchAllBrowseItems()
        let splitList = this.path.split('/')
        let title = splitList.pop()
        this.currentDirectory = splitList.join('/')
        if (this.mode === 'move') {
          this.currentTitle = title
        }
        this.currentLocale = this.locale
        _.delay(() => {
          this.$refs.pathIpt2.focus()
        })
      }
    },
    currentNode (newValue, oldValue) {
      if (newValue.length < 1) { // force a selection
        this.$nextTick(() => {
          this.currentNode = oldValue
        })
      } else {
        const current = _.find(this.all, ['id', newValue[0]])

        if (this.openNodes.indexOf(newValue[0]) < 0) { // auto open and load children
          if (current) {
            if (this.openNodes.indexOf(current.parent) < 0) {
              this.$nextTick(() => {
                this.openNodes.push(current.parent)
              })
            }
          }
          this.$nextTick(() => {
            this.openNodes.push(newValue[0])
          })
        }
        this.currentDirectory = _.get(current, 'path', '')
      }
    },
    currentPage (newValue, oldValue) {
      if (!_.isEmpty(newValue)) {
        this.currentPath = newValue.path
      }
    },
    currentLocale (newValue, oldValue) {
      this.$nextTick(() => {
        this.tree = [
          {
            id: 0,
            title: '/ (root)',
            children: []
          }
        ]
        this.currentNode = [0]
        this.openNodes = [0]
        this.pages = []
        this.all = []
        this.treeViewCacheId += 1
      })
    }
  },
  mounted () {
    // this.fetchAllBrowseItems()
  },
  methods: {
    close() {
      this.isShown = false
    },
    open() {
      const exit = this.openHandler({
        locale: this.currentLocale,
        path: this.currentPath,
        title: this.currentTitle,
        editor: this.currentEditor,
        id: (this.mustExist && this.currentPage) ? this.currentPage.pageId : 0
      })
      if (exit !== false) {
        this.close()
      }
    },
    async fetchAllBrowseItems() {
      this.$store.commit(`loadingStart`, 'browse-load')
      const resp = await this.$apollo.query({
        query: gql`
          query ($locale: String!) {
            pages {
              allTree(locale: $locale) {
                id
                path
                title
                isFolder
                pageId
                parent
                locale
              }
            }
          }
        `,
        fetchPolicy: 'cache-first',
        variables: {
          locale: this.locale
        }
      })
      const list = _.get(resp, 'data.pages.allTree', [])
      let array = []
      list.forEach(item => { // 遍历对象数组
        item.children = list.filter(info => info.parent === item.id)
        if (item.parent === 0) {
          array.push(item) // 将一层节点放入新数组中
        }
      })
      this.tree[0].children = array
      this.openNodes = [0]
      this.all = list
      this.$store.commit(`loadingStop`, 'browse-load')
    },
    async fetchFolders (item) {
      this.searchLoading = true
      const resp = await this.$apollo.query({
        query: gql`
          query ($parent: Int!, $mode: PageTreeMode!, $locale: String!) {
            pages {
              tree(parent: $parent, mode: $mode, locale: $locale) {
                id
                path
                title
                isFolder
                pageId
                parent
              }
            }
          }
        `,
        fetchPolicy: 'network-only',
        variables: {
          parent: item.id,
          mode: 'ALL',
          locale: this.currentLocale
        }
      })
      const items = _.get(resp, 'data.pages.tree', [])
      const itemFolders = _.filter(items, ['isFolder', true]).map(f => ({...f, children: []}))
      const itemPages = _.filter(items, i => i.pageId > 0)
      if (itemFolders.length > 0) {
        item.children = itemFolders
      } else {
        item.children = undefined
      }
      this.pages = _.unionBy(this.pages, itemPages, 'id')
      this.all = _.unionBy(this.all, items, 'id')

      this.searchLoading = false
    }
  }
}
</script>

<style lang='scss'>

.page-selector {
  .v-treeview-node__label {
    font-size: 13px;
  }
  .v-treeview-node__content {
    cursor: pointer;
  }
}

</style>
