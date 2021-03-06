<template>
  <div class="app normalize">
    <transition name="loading-horizontal-fade" v-if="!disableLoading">
      <loading-view v-if="!isDone && !isFailed" class="loading-view"></loading-view>
    </transition>
    <transition name="slow-horizontal-fade">
      <div class="failed-view" v-if="isFailed">
        <h4>{{ string.loadingFailedAndRefresh }}</h4>
        <article>
          <p>{{ string.failedMsg }}: {{ failedMsg }}</p>
          <p>{{ string.version}}: {{ config.version }}</p>
          <p>{{ string.ContractAuthor}}: Create issue on {{ config.homePage }} or email {{ config.email }}</p>
        </article>
      </div>
    </transition>
    <template v-if="isDone">
      <transition name="slow-horizontal-fade">
        <thumb-scroll-view
          v-if="supportThumbView"
          class="thumb-column"
          :style="thumbStyle"
          :thumbInfos="thumbInfos"
          :pageCount="pageCount"
        ></thumb-scroll-view>
      </transition>
      <transition name="slow-horizontal-fade">
        <reader-view
          class="reader-column"
          :pageCount="pageCount"
          :curPageNum="curPageNum"
          :title="title"
          :imgPageInfos="imgPageInfos"
          :albumId="albumId"
        ></reader-view>
      </transition>
    </template>
    <modal-manager class="modal"></modal-manager>
  </div>
</template>

<script>
import { mapGetters } from 'vuex';
import ThumbScrollView from './components/ThumbScrollView.vue';
import ReaderView from './components/ReaderView.vue';
import LoadingView from './components/LoadingView.vue';
import ModalManager from './components/ModalManager.vue';
import SettingService from './service/SettingService.ts';
import InfoService from './service/InfoService.ts';
import * as tags from './assets/value/tags';
import Utils from './utils/Utils.ts';

export default {
    name: 'InjectedApp',

    inject: ['config', 'service', 'disableLoading'],

    components: {
        ThumbScrollView,
        ReaderView,
        ModalManager,
        LoadingView
    },

    data() {
        return {
            pageCount: 0,
            curPageNum: 0,
            title: '',
            imgPageInfos: [],
            thumbInfos: [],
            albumId: '',
            isDone: false,
            isFailed: false,
            failedMsg: '',
            supportThumbView: false
        };
    },

    async created() {
        Promise.all([
            this.service.album.getPageCount(),
            this.service.album.getCurPageNum(),
            this.service.album.getTitle(),
            this.service.album.getImgPageInfos(),
            this.service.album.getThumbInfos(),
            this.service.album.getAlbumId()
        ]).then(
            async values => {
                this.pageCount = values[0];
                this.curPageNum = values[1];
                this.title = values[2];
                this.imgPageInfos = values[3];
                this.thumbInfos = values[4];
                this.albumId = values[5];
                this.supportThumbView = this.service.album.supportThumbView();
                this.isDone = true;
            },
            reason => {
                console.error('[Ehunter init error]', reason);
                this.failedMsg = reason;
                this.isFailed = true;
            }
        );
        await this.checkInstructions();
        this.checkVersion();
    },

    computed: {
        ...mapGetters(['showThumbView', 'thumbWidth', 'readingMode', 'showThumbViewInBook', 'string']),
        thumbStyle() {
            if (this.supportThumbView) {
                if (this.readingMode === 0 && this.showThumbView) {
                    return '';
                } else if (this.readingMode === 1 && this.showThumbViewInBook) {
                    return '';
                }
            }
            return { 'margin-left': this.px(-this.thumbWidth) };
        }
    },

    methods: {
        async checkInstructions() {
            if (await SettingService.getFirstOpen()) {
                // auto choose language
                let lang = navigator.language.toLowerCase();
                if (lang.includes('zh')) {
                    await SettingService.setLang(tags.LANG_CN);
                } else if (lang.includes('jp')) {
                    await SettingService.setLang(tags.LANG_JP);
                }
                // show instructions
                InfoService.showInstruction(this.config, true);
                setTimeout(() => SettingService.setFirstOpen(false), 2000);
            }
        },

        async checkVersion() {
            await Utils.timeout(5000); // avoid the reloading of storage update
            InfoService.checkUpdate(this.config);
            InfoService.checkNewVersion(this.config);
        }
    }
};
</script>

<style lang="scss">
@import './style/_responsive';
@import './style/_variables';
@import './style/_markdown';
@import './style/_normalize';

$change_mode_time: 0.8s;
$fast_change_mode_time: 0.4s;
$general_animtation_time: 0.2s;

.app {
    font-family: PingFang SC, Microsoft YaHei, 微软雅黑, Arial, Hiragino Sans GB, Heiti SC, Droid Sans,
        WenQuanYi Micro Hei, sans-serif !important;
    display: flex;
    height: 100%;
    text-align: initial; // overlay original style
    > .loading-view {
        position: absolute;
        top: 0;
        left: 0;
        bottom: 0;
        right: 0;
        z-index: 1;
    }

    > .failed-view {
        display: flex;
        flex: 1;
        flex-direction: column;
        justify-content: center;
        align-items: center;
        > h4 {
            color: #ffffff;
            text-align: center;
            text-transform: uppercase;
            font-size: 20px;
        }
        > article {
            > p {
                font-size: 12px;
                color: #eeeeee;
                font-size: 12px;
            }
        }
    }

    > .thumb-column {
        transition: all $change_mode_time ease;
        &.hide {
            margin: -100%;
        }
    }
    > .reader-column {
        flex: 1;
    }
    > .modal {
        position: fixed;
        top: 0;
        left: 0;
        right: 0;
        bottom: 0;
        z-index: 100;
        overflow-y: auto;
    }

    section,
    header,
    nav {
        display: flex;
    }

    p {
        padding: 0;
    }

    h1,
    h2,
    h3,
    h4,
    h5,
    h6 {
        margin: 0;
    }

    .clickable {
        cursor: pointer;
    }

    .no-select {
        user-select: none;
    }

    .tips {
        position: relative;
        &:hover {
            &:after {
                content: attr(title-content);
                position: absolute;
                top: -110%;
                left: 50%;
                transform: translate(-50%, 0);
                font-size: 12px;
                white-space: nowrap;
                padding: 4px 6px 5px 6px;
                border-radius: 2px;
                min-width: 50px;
                text-align: center;
                background: rgba(0, 0, 0, 0.8);
                box-shadow: 0 1px 6px rgba(0, 0, 0, 0.117647), 0 1px 4px rgba(0, 0, 0, 0.117647);
                color: white;
            }
        }
        &.tips-down {
            &:hover {
                &:after {
                    top: 130%;
                }
            }
        }
        &.tips-right {
            &:hover {
                &:after {
                    left: -10%;
                    transform: initial;
                }
            }
        }
        &.tips-left {
            &:hover {
                &:after {
                    right: -20%;
                    left: initial;
                    transform: initial;
                }
            }
        }
    }

    // 过渡效果
    .slide-fade-enter-active,
    .slide-fade-leave-active {
        transition: all $general_animtation_time ease;
    }
    .slide-fade-enter,
    .slide-fade-leave-active {
        transform: translateX(10px);
        opacity: 0;
    }

    .center-horizontal-fade-enter-active,
    .center-horizontal-fade-leave-active {
        transition: all $change_mode_time ease;
    }
    .center-horizontal-fade-enter,
    .center-horizontal-fade-leave-active {
        transform: translateX(-40%) !important;
        opacity: 0 !important;
    }

    .slow-horizontal-fade-enter-active,
    .slow-horizontal-fade-leave-active {
        transition: all $change_mode_time ease;
    }
    .slow-horizontal-fade-enter,
    .slow-horizontal-fade-leave-active {
        transform: translateX(20%);
        opacity: 0;
    }

    .loading-horizontal-fade-enter-active,
    .loading-horizontal-fade-leave-active {
        transition: all 0.5s ease;
    }
    .loading-horizontal-fade-enter,
    .loading-horizontal-fade-leave-active {
        transform: translateX(20%);
        opacity: 0;
    }

    .fast-horizontal-fade-enter-active,
    .fast-horizontal-fade-leave-active {
        transition: all $fast_change_mode_time ease;
    }
    .fast-horizontal-fade-enter,
    .fast-horizontal-fade-leave-active {
        transform: translateX(20%);
        opacity: 0;
    }

    .slow-vertical-fade-enter-active,
    .slow-vertical-fade-leave-active {
        transition: all $change_mode_time ease;
    }
    .slow-vertical-fade-enter,
    .slow-vertical-fade-leave-active {
        transform: translate(-20%, 20%);
        opacity: 0;
    }

    .slow-opacity-fade-enter-active,
    .slow-opacity-fade-leave-active {
        transition: all 0.3s ease;
    }
    .slow-opacity-fade-enter,
    .slow-opacity-fade-leave-active {
        opacity: 0;
    }

    .vertical-list-enter-active,
    .vertical-list-leave-active {
        transition: all 0.5s;
    }
    .vertical-list-enter,
    .vertical-list-leave-to {
        opacity: 0;
        transform: translateY(10%);
    }
}
</style>