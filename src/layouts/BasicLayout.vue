<template>
  <pro-layout
    :menus="menus"
    :collapsed="collapsed"
    :mediaQuery="query"
    :isMobile="isMobile"
    :handleMediaQuery="handleMediaQuery"
    :handleCollapse="handleCollapse"
    :i18nRender="i18nRender"
    :siderWidth="220"
    :collapsedWidth="50"
    v-bind="settings"
    :topMenu="topMenu"
    @top-menu-select="handleTopMenuSelect"
  >
    <!-- Ads begin
      广告代码 真实项目中请移除
      production remove this Ads
    -->
    <ads v-if="isProPreviewSite && !collapsed"/>
    <!-- Ads end -->

    <!-- 1.0.0+ 版本 pro-layout 提供 API，
          我们推荐使用这种方式进行 LOGO 和 title 自定义
    -->
    <template v-slot:menuHeaderRender>
      <div>
        <logo-svg />
        <h1>{{ title }}</h1>
      </div>
    </template>
    <!-- 1.0.0+ 版本 pro-layout 提供 API,
          增加 Header 左侧内容区自定义
    -->
    <!-- <template v-slot:headerContentRender>
      <div>
        <a-tooltip title="刷新页面">
          <a-icon type="reload" style="font-size: 18px;cursor: pointer;" @click="() => { $message.info('只是一个DEMO') }" />
        </a-tooltip>
      </div>
    </template> -->

    <setting-drawer :settings="settings" @change="handleSettingChange">
      <div style="margin: 12px 0;">
        This is SettingDrawer custom footer content.
      </div>
    </setting-drawer>
    <template v-slot:rightContentRender>
      <right-content :top-menu="settings.layout === 'topmenu'" :is-mobile="isMobile" :theme="settings.theme" />
    </template>
    <!-- custom footer / 自定义Footer -->
    <template v-slot:footerRender>
      <global-footer />
    </template>
    <router-view />
  </pro-layout>
</template>

<script>
import { SettingDrawer, updateTheme, RouteView } from '@hangar/pro-layout';
// import SettingDrawer from '@/components/SettingDrawer';
import { i18nRender } from '@/locales';
import { mapState } from 'vuex';
import { CONTENT_WIDTH_TYPE, SIDEBAR_TYPE, TOGGLE_MOBILE_TYPE, TOGGLE_MULTI_TAB } from '@/store/mutation-types';

import defaultSettings from '@/config/default-settings';
import RightContent from '@/components/GlobalHeader/RightContent';
import GlobalFooter from '@/components/GlobalFooter';
import Ads from '@/components/Other/CarbonAds';
import LogoSvg from '../assets/logo.svg?inline';
export default {
  name: 'BasicLayout',
  components: {
    SettingDrawer,
    RightContent,
    GlobalFooter,
    LogoSvg,
    Ads,
    RouteView
  },
  data() {
    return {
      // preview.pro.antdv.com only use.
      isProPreviewSite: process.env.VUE_APP_PREVIEW === 'true' && process.env.NODE_ENV !== 'development',
      // end

      // base
      menus: [],
      // 侧栏收起状态
      collapsed: false,
      title: defaultSettings.title,
      settings: {
        // 布局类型
        layout: defaultSettings.layout, // 'sidemenu', 'topmenu'
        // CONTENT_WIDTH_TYPE
        contentWidth: defaultSettings.layout === 'sidemenu' ? CONTENT_WIDTH_TYPE.Fluid : defaultSettings.contentWidth,
        // 主题 'dark' | 'light'
        theme: defaultSettings.navTheme,
        // 主色调
        primaryColor: defaultSettings.primaryColor,
        fixedHeader: defaultSettings.fixedHeader,
        fixSiderbar: defaultSettings.fixSiderbar,
        colorWeak: defaultSettings.colorWeak,

        hideHintAlert: false,
        hideCopyButton: false,
        multiTab: defaultSettings.multiTab,
        autoSplitMenus: defaultSettings.autoSplitMenus
      },
      // 媒体查询
      query: {},

      // 是否手机模式
      isMobile: false,
      topMenu: null
    };
  },
  computed: {
    ...mapState({
      // 动态主路由
      mainMenu: state => state.permission.addRouters,
      multiTab: state => state.app.multiTab
    })
  },
  created() {
    const routes = this.mainMenu.find(item => item.path === '/');
    this.menus = (routes && routes.children) || [];
    // 处理侧栏收起状态
    this.$watch('collapsed', () => {
      this.$store.commit(SIDEBAR_TYPE, this.collapsed);
    });
    this.$watch('isMobile', () => {
      this.$store.commit(TOGGLE_MOBILE_TYPE, this.isMobile);
    });
  },
  mounted() {
    const userAgent = navigator.userAgent;
    if (userAgent.indexOf('Edge') > -1) {
      this.$nextTick(() => {
        this.collapsed = !this.collapsed;
        setTimeout(() => {
          this.collapsed = !this.collapsed;
        }, 16);
      });
    }

    // first update color
    // TIPS: THEME COLOR HANDLER!! PLEASE CHECK THAT!!
    if (process.env.NODE_ENV !== 'production' || process.env.VUE_APP_PREVIEW === 'true') {
      updateTheme(this.settings.primaryColor);
    }
  },
  methods: {
    i18nRender,
    handleMediaQuery(val) {
      this.query = val;
      if (this.isMobile && !val['screen-xs']) {
        this.isMobile = false;
        return;
      }
      if (!this.isMobile && val['screen-xs']) {
        this.isMobile = true;
        this.collapsed = false;
        this.settings.contentWidth = CONTENT_WIDTH_TYPE.Fluid;
        // this.settings.fixSiderbar = false
      }
    },
    handleCollapse(val) {
      this.collapsed = val;
    },
    handleSettingChange({ type, value }) {
      type && (this.settings[type] = value);
      switch (type) {
        case 'contentWidth':
          this.settings[type] = value;
          break;
        case 'layout':
          if (value === 'sidemenu' || value === 'mix') {
            this.settings.contentWidth = CONTENT_WIDTH_TYPE.Fluid;
          } else {
            this.settings.fixSiderbar = false;
            this.settings.contentWidth = CONTENT_WIDTH_TYPE.Fluid;
          }
          break;
        case 'multiTab':
          this.$store.commit(TOGGLE_MULTI_TAB, value);
          break;
      }
    },
    handleTopMenuSelect(menu) {
      this.topMenu = menu;
    }
  }
};
</script>

<style lang="less">
@import "./BasicLayout.less";
</style>
