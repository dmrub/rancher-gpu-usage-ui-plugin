<script>
import Loading from '@shell/components/Loading';
import { Banner } from '@components/Banner';

export default {
  components: { Banner, Loading },
  props:      {
    url: {
      type:     String,
      required: true,
    },
    vars: {
      type:    Object,
      default: () => ({})
    },
    range: {
      type:    String,
      default: null
    },
    refreshRate: {
      type:    String,
      default: null
    },
    backgroundColor: {
      type:    String,
      default: '#1b1c21'
    },
    theme: {
      type:    String,
      default: 'dark'
    }
  },
  async fetch() {
    //const inStore = this.$store.getters['currentProduct'].inStore;
    //const res = await this.$store.dispatch(`${ inStore }/find`, { type: CATALOG.APP, id: 'cattle-monitoring-system/rancher-monitoring' });

    //this.monitoringVersion = res?.currentVersion;
  },
  data() {
    return {
      loading: false, error: false, interval: null, errorTimer: null
    };
  },
  computed: {
    currentUrl() {
      return this.computeUrl();
    },
    gpuUsageUrl() {
      return this.currentUrl.replace('&kiosk', '');
    },
    gpuUsageWindow() {
      return this.$refs.frame?.contentWindow;
    },
    graphHistory() {
      return this.gpuUsageWindow?.history;
    },
    gpuUsageDocument() {
      return this.gpuUsageWindow?.document;
    }
  },
  watch: {
    currentUrl(neu) {
      // Should consider changing `this.gpuUsageWindow?.angular` to something like `!loaded && !error`
      // https://github.com/rancher/dashboard/pull/5802
      if (this.graphHistory && this.gpuUsageWindow?.angular) {
        this.gpuUsageWindow.location.replace(neu);
      }
    },

    error(neu) {
      if (neu) {
        this.errorTimer = setInterval(() => {
          this.reload();
        }, 45000);
      } else {
        clearInterval(this.errorTimer);
        this.errorTimer = null;
      }
    }
  },
  mounted() {
    this.$refs.frame.onload = this.inject;
    this.poll();
  },
  beforeDestroy() {
    if (this.interval) {
      clearInterval(this.interval);
    }

    if (this.errorTimer) {
      clearInterval(this.errorTimer);
    }
  },
  methods: {
    poll() {
      if (this.interval) {
        clearInterval(this.interval);
        this.interval = null;
      }

      this.interval = setInterval(() => {
        try {
          const gpuUsageWindow = this.$refs.frame?.contentWindow;
          /*
          const errorElements = gpuUsageWindow.document.getElementsByClassName('alert-error');
          const errorCornerElements = gpuUsageWindow.document.getElementsByClassName('panel-info-corner--error');
          const panelInFullScreenElements = gpuUsageWindow.document.getElementsByClassName('panel-in-fullscreen');
          const panelContainerElements = gpuUsageWindow.document.getElementsByClassName('panel-container');
          const error = errorElements.length > 0 || errorCornerElements.length > 0;
          const loaded = panelInFullScreenElements.length > 0 || panelContainerElements.length > 0;
          const errorMessageElms = gpuUsageWindow.document.getElementsByTagName('pre');
          const errorMessage = errorMessageElms.length > 0 ? errorMessageElms[0].innerText : '';
          const isFailure = errorMessage.includes('"status": "Failure"');
          */
          const error = false;
          const loaded = true;
          const isFailure = false;

          if (error) {
            throw new Error('An error was detected in the iframe');
          }

          this.$set(this, 'loading', !loaded);
          this.$set(this, 'error', isFailure);
        } catch (ex) {
          this.$set(this, 'error', true);
          this.$set(this, 'loading', false);
          clearInterval(this.interval);
          this.interval = null;
        }
      }, 100);
    },
    computeFromTo() {
      return {
        from: `now-${ this.range }`,
        to:   `now`
      };
    },
    computeUrl() {
      const embedUrl = this.url;
      const clusterId = this.$store.getters['currentCluster'].id;

      return `/k8s/clusters/${ clusterId }/api/v1/namespaces/kube-utils/services/http:k8s-gpu-usage:80/proxy`;
    },
    reload(ev) {
      ev && ev.preventDefault();
      this.$refs.frame.contentWindow.location.reload();
      this.poll();
    },
    injectCss() {
      const style = document.createElement('style');

      style.innerHTML = `
        body .gpu-usage-app .dashboard-content {
          background: ${ this.backgroundColor };
          padding: 0;
        }

        body .gpu-usage-app .layout {
          background: ${ this.backgroundColor };
        }


        body .gpu-usage-app .dashboard-content .panel-container {
          background-color: initial;
          border: none;
        }

        body .gpu-usage-app .dashboard-content .panel-wrapper {
          height: 100%;
        }

        body .gpu-usage-app .panel-menu-container {
          display: none;
        }

        body .gpu-usage-app .panel-title {
          cursor: default;
        }

        body .gpu-usage-app .panel-title .panel-title-text div {
          display: none;
        }
      `;

      const gpuUsageWindow = this.$refs.frame?.contentWindow;
      const gpuUsageDocument = gpuUsageWindow?.document;

      if (gpuUsageDocument.head) {
        gpuUsageDocument.head.appendChild(style);
      }
    },

    inject() {
      this.injectCss();
    }
  }
};
</script>

<template>
  <div class="gpu-usage-dashboard">
    <Banner
      v-if="error"
      color="error"
      style="z-index: 1000"
    >
      <div class="text-center">
        {{ t('gpu-usage.failedToLoad') }} <a
          href="#"
          @click="reload"
        >{{ t('gpu-usage.reload') }}</a>
      </div>
    </Banner>
    <iframe
      v-show="!error"
      ref="frame"
      :class="{loading, frame: true}"
      :src="currentUrl"
      frameborder="0"
      scrolling="yes"
    />
    <div v-if="loading">
      <Loading />
    </div>
    <div
      v-if="!loading && !error"
      class="external-link"
    >
      <!-- https://github.com/harvester/harvester-installer/pull/512/files -->
      <!-- It is necessary to include the parameter referer when accessing the Grafana page. -->
      <!-- This parameter is required by the backend to identify the origin of the request from which cluster -->
      <!-- The matching mechanism as follows: -->
      <!-- ~.*/k8s/clusters/(c-m-.+)/.* -->
      <!-- ~.*/dashboard/harvester/c/(c-m-.+)/.* -->
      <a
        :href="gpuUsageUrl"
        target="_blank"
        rel="noopener nofollow"
      >{{ t('gpu-usage.description') }} <i class="icon icon-external-link" /></a>
    </div>
  </div>
</template>

<style lang='scss' scoped>
.gpu-usage-dashboard {
  position: relative;
  min-height: 100%;
  min-width: 100%;

  & ::v-deep .content {
    position: relative;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    width: 100%;
    height: 100%;
    padding: 0;
  }

  & ::v-deep .overlay {
    position: static;
    background-color: initial;
  }

  iframe {
    position: absolute;
    left: 0;
    right: 0;
    top: 20px;
    bottom: 0;
    width: 100%;
    height: 100%;
    overflow: hidden;

    &.loading {
      visibility: hidden;
    }
  }
}
</style>
