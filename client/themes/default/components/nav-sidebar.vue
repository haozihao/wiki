<template lang="pug">
  div
    .pa-3.d-flex(v-if='navMode === `MIXED`', :class='$vuetify.theme.dark ? `grey darken-5` : `blue darken-3`')
      v-btn(
        depressed
        :color='$vuetify.theme.dark ? `grey darken-4` : `blue darken-2`'
        style='min-width:0;'
        @click='goHome'
        :aria-label='$t(`common:header.home`)'
        )
        v-icon(size='20') mdi-home
      v-btn.ml-3(
        v-if='currentMode === `custom`'
        depressed
        :color='$vuetify.theme.dark ? `grey darken-4` : `blue darken-2`'
        style='flex: 1 1 100%;'
        @click='switchMode(`browse`)'
        )
        v-icon(left) mdi-file-tree
        .body-2.text-none {{$t('common:sidebar.browse')}}
      v-btn.ml-3(
        v-else-if='currentMode === `browse`'
        depressed
        :color='$vuetify.theme.dark ? `grey darken-4` : `blue darken-2`'
        style='flex: 1 1 100%;'
        @click='switchMode(`custom`)'
        )
        v-icon(left) mdi-navigation
        .body-2.text-none {{$t('common:sidebar.mainMenu')}}
    v-divider
    //-> Custom Navigation
    v-list.py-2(v-if='currentMode === `custom`', dense, :class='color', :dark='dark')
      template(v-for='item of items')
        v-list-item(
          v-if='item.k === `link`'
          :href='item.t'
          :target='item.y === `externalblank` ? `_blank` : `_self`'
          :rel='item.y === `externalblank` ? `noopener` : ``'
          )
          v-list-item-avatar(size='24', tile)
            v-icon(v-if='item.c.match(/fa[a-z] fa-/)', size='19') {{ item.c }}
            v-icon(v-else) {{ item.c }}
          v-list-item-title {{ item.l }}
        v-divider.my-2(v-else-if='item.k === `divider`')
        v-subheader.pl-4(v-else-if='item.k === `header`') {{ item.l }}
    //-> Browse
    v-list.py-2(v-else-if='currentMode === `browse`', dense, :class='color', :dark='dark')
      template
        v-treeview.page-tree(
          v-model="tree"
          :items="allBrowseTree"
          item-key="id"
          item-text="title"
          :open.sync="openTree"
          :active="activeTree"
          @update:active="updateActive"
          activatable
          dense
          color
          hoverable)
          template(v-slot:prepend="{item}")
            v-icon(v-if='item.isFolder') mdi-folder
            v-icon(v-else) mdi-text-box
</template>

<script>
import _ from 'lodash'
import gql from 'graphql-tag'
import { get } from 'vuex-pathify'

/* global siteLangs */

export default {
  props: {
    color: {
      type: String,
      default: 'primary'
    },
    dark: {
      type: Boolean,
      default: true
    },
    items: {
      type: Array,
      default: () => []
    },
    navMode: {
      type: String,
      default: 'MIXED'
    }
  },
  data() {
    return {
      tree: [],
      openTree: [],
      activeTree: [],
      allBrowseList: [],
      allBrowseTree: [],
      currentMode: 'custom',
      currentItems: [],
      currentParent: {
        id: 0,
        title: '/ (root)'
      },
      parents: [],
      loadedCache: []
    }
  },
  computed: {
    path: get('page/path'),
    locale: get('page/locale')
  },
  methods: {
    switchMode (mode) {
      this.currentMode = mode
      window.localStorage.setItem('navPref', mode)
      if (mode === `browse` && this.loadedCache.length < 1) {
        this.loadFromCurrentPath()
      }
    },
    updateActive(obj) {
      sessionStorage.setItem('activeTree', JSON.stringify(obj))
      if (obj.length > 0) {
        const activeItem = this.allBrowseList.find(item => item.id === obj[0])
        if (obj[0] !== this.activeTree[0]) {
          sessionStorage.setItem('openTree', JSON.stringify(this.openTree))
          // this.$router.push({ path: `/` + item.locale + `/` + item.path })
          window.location.assign(`/` + activeItem.locale + `/` + activeItem.path)
        }
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
      this.allBrowseList = list
      this.allBrowseTree = array
      const oldActiveTree = JSON.parse(sessionStorage.getItem('activeTree'))
      const oldOpenTree = JSON.parse(sessionStorage.getItem('openTree'))
      this.activeTree = oldActiveTree || []
      this.openTree = oldOpenTree || []
      this.$store.commit(`loadingStop`, 'browse-load')
    },
    async fetchBrowseItems (item) {
      this.$store.commit(`loadingStart`, 'browse-load')
      if (!item) {
        item = this.currentParent
      }

      if (this.loadedCache.indexOf(item.id) < 0) {
        this.currentItems = []
      }

      if (item.id === 0) {
        this.parents = []
      } else {
        const flushRightIndex = _.findIndex(this.parents, ['id', item.id])
        if (flushRightIndex >= 0) {
          this.parents = _.take(this.parents, flushRightIndex)
        }
        if (this.parents.length < 1) {
          this.parents.push(this.currentParent)
        }
        this.parents.push(item)
      }

      this.currentParent = item

      const resp = await this.$apollo.query({
        query: gql`
          query ($parent: Int, $locale: String!) {
            pages {
              tree(parent: $parent, mode: ALL, locale: $locale) {
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
          parent: item.id,
          locale: this.locale
        }
      })
      this.loadedCache = _.union(this.loadedCache, [item.id])
      this.currentItems = _.get(resp, 'data.pages.tree', [])
      this.$store.commit(`loadingStop`, 'browse-load')
    },
    async loadFromCurrentPath() {
      this.$store.commit(`loadingStart`, 'browse-load')
      const resp = await this.$apollo.query({
        query: gql`
          query ($path: String, $locale: String!) {
            pages {
              tree(path: $path, mode: ALL, locale: $locale, includeAncestors: true) {
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
          path: this.path,
          locale: this.locale
        }
      })
      const items = _.get(resp, 'data.pages.tree', [])
      const curPage = _.find(items, ['pageId', this.$store.get('page/id')])
      if (!curPage) {
        console.warn('Could not find current page in page tree listing!')
        return
      }

      let curParentId = curPage.parent
      let invertedAncestors = []
      while (curParentId) {
        const curParent = _.find(items, ['id', curParentId])
        if (!curParent) {
          break
        }
        invertedAncestors.push(curParent)
        curParentId = curParent.parent
      }

      this.parents = [this.currentParent, ...invertedAncestors.reverse()]
      this.currentParent = _.last(this.parents)

      this.loadedCache = [curPage.parent]
      this.currentItems = _.filter(items, ['parent', curPage.parent])
      this.$store.commit(`loadingStop`, 'browse-load')
    },
    goHome () {
      sessionStorage.setItem('activeTree', JSON.stringify([]))
      sessionStorage.setItem('openTree', JSON.stringify([]))
      window.location.assign(siteLangs.length > 0 ? `/${this.locale}/home` : '/')
    }
  },
  mounted () {
    this.currentParent.title = `/ ${this.$t('common:sidebar.root')}`
    if (this.navMode === 'TREE') {
      this.currentMode = 'browse'
    } else if (this.navMode === 'STATIC') {
      this.currentMode = 'custom'
    } else {
      this.currentMode = window.localStorage.getItem('navPref') || 'custom'
    }
    if (this.currentMode === 'browse') {
      this.loadFromCurrentPath()
    }
    this.fetchAllBrowseItems()
  }
}
</script>

<style lang='scss'>

.page-tree {
  .v-treeview-node__content {
    cursor: pointer;
  }
}

</style>
