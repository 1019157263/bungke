<template>
  <div class="tags-view-container">
    <scroll-pane ref="scrollPane" class="tags-view-wrapper">
      <router-link
        v-for="tag in visitedViews"
        ref="tag"
        :class="isActive(tag)?'active':''"
        :to="{ path: tag.path, query: tag.query, fullPath: tag.fullPath }"
        :key="tag.fullPath"
        tag="span"
        @click.native="onRouterLinkPressed(tag)"
        class="tags-view-item"
        @click.middle.native="closeSelectedTag(tag)"
        @contextmenu.prevent.native="openMenu(tag,$event)">
        {{ tag.title }}
        <span class="el-icon-close" @click.prevent.stop="closeSelectedTag(tag)"/>
      </router-link>
    </scroll-pane>
    <ul v-show="visible" :style="{left:left+'px',top:top+'px'}" class="contextmenu">
      <!--<li @click="refreshSelectedTag(selectedTag)">刷新</li>-->
      <li @click="closeSelectedTag(selectedTag)">关闭</li>
      <li @click="closeOthersTags">关闭其他</li>
      <li @click="closeAllTags">关闭所有</li>
    </ul>
  </div>
</template>

<script>
  import ScrollPane from "@/components/ScrollPane"

  export default {
    components: { ScrollPane },
    data () {
      return {
        visible: false,
        top: 0,
        left: 0,
        selectedTag: {}
      }
    },
    computed: {
      visitedViews () {
        return this.$store.state.tagsView.visitedViews
      }
    },
    watch: {
      $route () {
        this.addViewTags()
        this.moveToCurrentTag()
      },
      visible (value) {
        if (value) {
          document.body.addEventListener("click", this.closeMenu)
        } else {
          document.body.removeEventListener("click", this.closeMenu)
        }
      }
    },
    mounted () {
      this.addViewTags()
    },
    methods: {
      onRouterLinkPressed (tag) {
        this.$store.commit("ADD_IFRAME", tag.query.url)
        this.$store.commit("SET_CURRENT_IFRAME_URL", tag.query.url)
      },
      isActive (route) {
        return route.fullPath === this.$route.fullPath
      },
      addViewTags () {
        const { name } = this.$route
        if (name) {
          this.$store.dispatch("addView", this.$route)
        } else {
          console.error("添加TagsView，名字为空，添加失败，route = ", this.$route)
        }
        return false
      },
      moveToCurrentTag () {
        const tags = this.$refs.tag
        this.$nextTick(() => {
          for (const tag of tags) {
            if (tag.to.fullPath === this.$route.fullPath) {
              this.$refs.scrollPane.moveToTarget(tag)

              // when query is different then update
              if (tag.to.fullPath !== this.$route.fullPath) {
                this.$store.dispatch("updateVisitedView", this.$route)
              }

              break
            }
          }
        })
      },
      // refreshSelectedTag (view) {
      //   this.$store.dispatch("delCachedView", view).then(() => {
      //     const { fullPath } = view
      //     this.$nextTick(() => {
      //       this.$router.push(fullPath)
      //       // this.$router.go(0)
      //     })
      //   })
      // },
      showLastTagTip () {
        this.$message.warning("这是最后一页了哦，不能关闭了")
      },
      closeSelectedTag (view) {
        if (this.visitedViews.length === 1) {
          this.showLastTagTip()
          return
        }
        if (view.fullPath.startsWith("/iframe?")) {
          this.$store.commit("REMOVE_IFRAME", view.query.url)
        }
        this.$store.dispatch("delView", view).then(({ visitedViews }) => {
          if (this.isActive(view)) { // 关闭的当前tag
            const latestView = visitedViews.slice(-1)[0]
            if (latestView) {
              this.$router.push(latestView.fullPath)
              if (latestView.fullPath.startsWith("/iframe?")) {
                this.$store.commit("SET_CURRENT_IFRAME_URL", latestView.query.url)
              }
              this.$store.commit("ADD_IFRAME", latestView.query.url)
              this.$store.commit("SET_CURRENT_IFRAME_URL", latestView.query.url)
            } else {
              this.$router.push("/")
            }
          }
        })
      },
      closeOthersTags () {
        this.$router.push(this.selectedTag.fullPath)
        if (this.selectedTag.fullPath.startsWith("/iframe?")) {
          this.$store.commit("REMOVE_OTHER_IFRAME", this.selectedTag.query.url)
        }
        this.$store.dispatch("delOthersViews", this.selectedTag).then(() => {
          this.moveToCurrentTag()
        })
      },
      closeAllTags () {
        if (this.visitedViews.length === 1) {
          this.showLastTagTip()
          return
        }
        if (this.$route.path === "/") { // 如果当前页面就是首页，直接push("/")是不会刷新的
          this.closeOthersTags()
          return
        }
        this.$store.commit("REMOVE_ALL_IFRAME")
        this.$store.dispatch("delAllViews")
        this.$router.push("/")
      },
      openMenu (tag, e) {
        const menuMinWidth = 105
        const offsetLeft = this.$el.getBoundingClientRect().left // container margin left
        const offsetWidth = this.$el.offsetWidth // container width
        const maxLeft = offsetWidth - menuMinWidth // left boundary
        const left = e.clientX - offsetLeft + 15 // 15: margin right

        if (left > maxLeft) {
          this.left = maxLeft
        } else {
          this.left = left
        }
        this.top = e.clientY

        this.visible = true
        this.selectedTag = tag
      },
      closeMenu () {
        this.visible = false
      }
    }
  }
</script>

<style rel="stylesheet/scss" lang="scss" scoped>
  .tags-view-container {
    height: 34px;
    width: 100%;
    background: #fff;
    border-bottom: 1px solid #d8dce5;
    box-shadow: 0 1px 3px 0 rgba(0, 0, 0, .12), 0 0 3px 0 rgba(0, 0, 0, .04);

    .tags-view-wrapper {
      .tags-view-item {
        display: inline-block;
        position: relative;
        cursor: pointer;
        height: 26px;
        line-height: 26px;
        border: 1px solid #d8dce5;
        color: #495060;
        background: #fff;
        padding: 0 8px;
        font-size: 12px;
        margin-left: 5px;
        margin-top: 4px;

        &:first-of-type {
          margin-left: 15px;
        }

        &:last-of-type {
          margin-right: 15px;
        }

        &.active {
          background-color: #42b983;
          color: #fff;
          border-color: #42b983;

          &::before {
            content: '';
            background: #fff;
            display: inline-block;
            width: 8px;
            height: 8px;
            border-radius: 50%;
            position: relative;
            margin-right: 2px;
          }
        }
      }
    }

    .contextmenu {
      margin: 0;
      background: #fff;
      z-index: 100;
      position: absolute;
      list-style-type: none;
      padding: 5px 0;
      border-radius: 4px;
      font-size: 12px;
      font-weight: 400;
      color: #333;
      box-shadow: 2px 2px 3px 0 rgba(0, 0, 0, .3);

      li {
        margin: 0;
        padding: 7px 16px;
        cursor: pointer;

        &:hover {
          background: #eee;
        }
      }
    }
  }
</style>

<style rel="stylesheet/scss" lang="scss">
  //reset element css of el-icon-close
  .tags-view-wrapper {
    .tags-view-item {
      .el-icon-close {
        width: 16px;
        height: 16px;
        vertical-align: 2px;
        border-radius: 50%;
        text-align: center;
        transition: all .3s cubic-bezier(.645, .045, .355, 1);
        transform-origin: 100% 50%;

        &:before {
          transform: scale(.6);
          display: inline-block;
          vertical-align: -3px;
        }

        &:hover {
          background-color: #b4bccc;
          color: #fff;
        }
      }
    }
  }
</style>
