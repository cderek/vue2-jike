<template>
  <div class="hot page-wrap" transition-mode="out-in">
    <nv-head page-type="hot" :fix-head="true" :show-menu="false" :need-add="true"></nv-head>
    <section>
      <div class="msg" v-for="item in topics" :key="item.id">
        <div class="msg-header">
          <img class="msg-topic-img" :src="item.topic.thumbnailUrl" />
          <router-link :to="{name:'topic',params:{id:item.id}}">
            <h1 class="topic-title">{{item.title}}</h1>
            <p>
              <time>{{item.createdAt | getLastTimeStr(true)}}</time>
            </p>
          </router-link>
        </div>
        <div class="message-content">
          <p class="content-text content">
            <span> {{item.content}} </span>
          </p>
          <div v-if="item.pictureUrls.length === 1" class="image-list single">
            <div class="image-list-content clear">
              <div class="image-item">
                <div class="image-wrap fixed-ratio" style="padding-top: 56.25%;">
                  <div class="image-border">
                    <div class="image" lazy="loaded" :style="{ 'background-image': 'url(' + item.pictureUrls[0].middlePicUrl + ')' }">
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
          <div v-if="item.pictureUrls.length != 0 && item.pictureUrls.length !=1" class="image-list">
            <div class="image-list-content clear">
              <div class="image-item" v-for="pictureUrl in item.pictureUrls">
                <div class="image-wrap fixed-ratio" style="padding-top: 100%;">
                  <div class="image-border">
                    <div v-if="pictureUrl.thumbnailUrl" class="image" lazy="loaded" :style="{ 'background-image': 'url(' + pictureUrl.thumbnailUrl + ')' }"></div>
                  </div>
                </div>
              </div>
            </div>
          </div>
          <div v-if="item.sourceRawValue === 'video'" class="video-player msg-video is-bilibili">
            <div v-if="item.video" class="wrap" lazy="loaded" :style="{ 'background-image': 'url(' + item.video.thumbnailUrl + ')' }">
              <vue-video></vue-video>
              <video :src="item.videoLink" controls="controls" preload="true" playsinline="true">
                您的浏览器不支持 video 标签。
              </video>
              <div v-if="item.video.thumbnailUrl" class="cover" lazy="loaded" :style="{ 'background-image': 'url(' + item.video.thumbnailUrl + ')' }">
                <div class="btn-play">
                  <img src="data:image/svg+xml;base64,PHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHdpZHRoPSI0MCIgaGVpZ2h0PSI0MCIgdmlld0JveD0iMCAwIDQwIDQwIj4KICAgIDxnIGZpbGw9Im5vbmUiIGZpbGwtcnVsZT0iZXZlbm9kZCI+CiAgICAgICAgPGNpcmNsZSBjeD0iMjAiIGN5PSIyMCIgcj0iMjAiIGZpbGw9IiNGRkU0MTEiLz4KICAgICAgICA8cGF0aCBmaWxsPSIjRkZGIiBmaWxsLXJ1bGU9Im5vbnplcm8iIGQ9Ik0xNSAxNC41NTVjMC0xLjUxNyAxLjA5LTIuMTMxIDIuMzk3LTEuMzczbDkuNDggNS40NTZjMS4zMDcuNzU5IDEuMzA3IDEuOTg3IDAgMi43NDZsLTkuNDggNS40NTZjLTEuMzA3LjcyMy0yLjM5Ny4xMDgtMi4zOTctMS4zNzNWMTQuNTU1eiIvPgogICAgPC9nPgo8L3N2Zz4K" alt="">
                </div>
                <span class="duration">{{ item.video.duration | date('%m:%s') }}</span>
              </div>
            </div>
          </div>
        </div>
        <router-link :to="{ name: 'message', params: { id:item.id }}" class="msg-footer">
          <div class="msg-footer-left">
            {{item.likeCount}}点赞 · {{item.commentCount}}评论 · {{item.repostCount}}转发
          </div>
          <div class="msg-footer-right">详情</div>
        </router-link>
      </div>
    </section>
    <div class="loading" v-if="!topics.length&&!refreshing">
      <span class="left"></span>
      <span class="middle"></span>
      <span class="right"></span>
    </div>
    <infinite-scroll :scroller="scroller" :loading="loading" @load="loadMore" />
    <nv-foot></nv-foot>
  </div>
</template>
<script>
import $ from 'webpack-zepto'
import {messageRecommend} from '@/service/api'
import nvHead from '@/components/header.vue'
import vueVideo from '@/components/vue-video.vue'
import nvFoot from '@/components/footer.vue'
import {getLastTimeStr} from '@/lib/utils.js'

export default {
  filters: {
    getLastTimeStr (time, isFromNow) {
      return getLastTimeStr(time, isFromNow)
    }
  },
  data () {
    return {
      refreshing: false,
      loading: false,
      count: 1,
      scroller: null,
      topics: [],
      index: {},
      searchKey: {
        page: 1,
        limit: 20,
        tab: 'all',
        mdrender: true
      },
      searchDataStr: ''
    }
  },
  mounted () {
    this.scroller = this.$el
    if (this.$route.query && this.$route.query.tab) {
      this.searchKey.tab = this.$route.query.tab
    }
    if (window.sessionStorage.searchKey && window.sessionStorage.tab === this.searchKey.tab) {
      this.topics = JSON.parse(window.sessionStorage.topics)
      this.searchKey = JSON.parse(window.sessionStorage.searchKey)
      this.$nextTick(() => $(window).scrollTop(window.sessionStorage.scrollTop))
    } else {
      this.getTopics()
    }
  },
  beforeRouteLeave (to, from, next) {
    if (to.name === 'topic') {
      window.sessionStorage.scrollTop = $(window).scrollTop()
      window.sessionStorage.topics = JSON.stringify(this.topics)
      window.sessionStorage.searchKey = JSON.stringify(this.searchKey)
      window.sessionStorage.tab = from.query.tab || 'all'
    }
    $(window).off('scroll')
    next()
  },
  beforeRouteEnter (to, from, next) {
    if (from.name !== 'topic') {
      if (window.sessionStorage.tab) {
        window.sessionStorage.removeItem('topics')
        window.sessionStorage.removeItem('searchKey')
        window.sessionStorage.removeItem('tab')
      }
    }
    next()
  },
  methods: {
    getTopics () {
      messageRecommend().then(res => {
        if (res && res.data) {
          res.data.forEach(this.mergeTopics)
          this.loading = false
        }
      })
    },
    mergeTopics (topic) {
      if (typeof this.index[topic.id] === 'number') {
        const topicsIndex = this.index[topic.id]
        this.topics[topicsIndex] = topic
      } else {
        this.index[topic.id] = this.topics.length
        this.topics.push(topic)
      }
    },
    loadMore () {
      this.loading = true
      setTimeout(() => {
        this.count--
        this.getTopics()
      }, 500)
    }
  },
  watch: {
    '$route' (to, from) {
      if (to.query && to.query.tab) {
        this.searchKey.tab = to.query.tab
        this.topics = []
        this.index = {}
      }
    }
  },
  components: {
    nvHead,
    vueVideo,
    nvFoot
  }
}
</script>
