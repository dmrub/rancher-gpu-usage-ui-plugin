<script>
import LazyImage from '@shell/components/LazyImage';

const VERSION = '0.2.0';

export default {
  name:       'GPUUsageDashboardCard',
  components: { LazyImage },
  props:      {
    resource: {
      type:     Object,
      required: true
    }
  },
  async fetch() {
    await this.fetchDeps();
  },
  data() {
    const gpuCardSrc = require('@pkg/gpu-usage/icons/gpu-card.svg');
    const currentCluster = this.$store.getters['currentCluster'];

    return {
      version:       VERSION,
      externalLinks: [
        {
          enabled:     false,
          iconSrc:     gpuCardSrc,
          kind:        'gpu-usage',
          description: 'gpu-usage.linkedList.description',
          label:       'gpu-usage.linkedList.label',
          link:        `/k8s/clusters/${ currentCluster.id }/api/v1/namespaces/kube-utils/services/http:k8s-gpu-usage:80/proxy`,
          status:      ''
        }
      ]
    };
  },
  methods: {

    async fetchDeps() {
      const { externalLinks } = this;

      for (const externalLink of externalLinks) {
        try {
          const response = await this.$axios.get(externalLink.link, { timeout: 5000 });

          if (response) {
            if (response.status === 200) {
              externalLink.enabled = true;
            } else {
              externalLink.status = `Could not get link, status code: ${ response.status.toString() }`;
            }
          } else {
            externalLink.status = 'Could not get link, no status code';
          }
        } catch (e) {
          externalLink.status = `Could not get link: ${ e.toString() }`;
        }

        if (externalLink.status) {
          externalLink.status = `. ${ externalLink.status }`;
        }
      }
    }
  }
};
</script>
<!-- !fel.enabled ? t('gpu-usage.linkedList.na') + fel.status : undefined -->
<template>
  <div>
    <div class="create-resource-container">
      <div class="subtypes-container">
        <a
          v-for="fel in externalLinks"
          :key="fel.label"
          v-clean-tooltip="
            !fel.enabled ? t('gpu-usage.linkedList.na') : undefined
          "
          :href="fel.enabled ? fel.link : void 0"
          :disabled="!fel.enabled"
          target="_blank"
          rel="noopener noreferrer"
          :class="{ 'subtype-banner': true, disabled: !fel.enabled }"
        >
          <div class="subtype-content">
            <div class="title">
              <div class="subtype-logo">
                <LazyImage :src="fel.iconSrc" />
              </div>
              <h5>
                <span>
                  <t :k="fel.label" />
                </span>
              </h5>
              <div class="flex-right">
                <i class="icon icon-external-link" />
              </div>
            </div>
            <hr>
            <div class="description">
              <span>
                <t :k="fel.description" />
              </span>
            </div>
          </div>
        </a>
      </div>
    </div>
  </div>
</template>

<style lang="scss" scoped>
  .create-resource-container .subtype-banner {
    min-height: 80px;
    padding: 10px;
  }
</style>
