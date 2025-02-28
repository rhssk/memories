<template>
  <div class="top-matter">
    <NcBreadcrumbs>
      <NcBreadcrumb :name="rootFolderName" :to="getRoute([])">
        <template #icon>
          <template v-if="routeIsPublic">
            <ShareIcon :size="20" />
            <span class="share-name">{{ rootFolderName }}</span>
          </template>
          <template v-else>
            <HomeIcon :size="20" />
          </template>
        </template>
      </NcBreadcrumb>
      <NcBreadcrumb v-for="folder in list" :key="folder.idx" :name="folder.text" :to="getRoute(folder.path)" />
    </NcBreadcrumbs>

    <div class="right-actions">
      <NcActions :inline="2">
        <NcActionButton @click="toggleRecursive" close-after-click>
          {{ recursive ? t('memories', 'Folder View') : t('memories', 'Timeline View') }}
          <template #icon>
            <FoldersIcon v-if="recursive" :size="20" />
            <TimelineIcon v-else :size="20" />
          </template>
        </NcActionButton>
        <NcActionButton
          v-if="!routeIsPublic"
          :aria-label="t('memories', 'Share folder')"
          @click="share()"
          close-after-click
        >
          {{ t('memories', 'Share folder') }}
          <template #icon> <ShareIcon :size="20" /> </template>
        </NcActionButton>
      </NcActions>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue';

import UserConfig from '@mixins/UserConfig';

const NcBreadcrumbs = () => import('@nextcloud/vue/dist/Components/NcBreadcrumbs');
const NcBreadcrumb = () => import('@nextcloud/vue/dist/Components/NcBreadcrumb');
import NcActions from '@nextcloud/vue/dist/Components/NcActions';
import NcActionButton from '@nextcloud/vue/dist/Components/NcActionButton';

import * as utils from '@services/utils';

import HomeIcon from 'vue-material-design-icons/Home.vue';
import ShareIcon from 'vue-material-design-icons/ShareVariant.vue';
import TimelineIcon from 'vue-material-design-icons/ImageMultiple.vue';
import FoldersIcon from 'vue-material-design-icons/FolderMultiple.vue';

export default defineComponent({
  name: 'FolderTopMatter',

  components: {
    NcBreadcrumbs,
    NcBreadcrumb,
    NcActions,
    NcActionButton,
    HomeIcon,
    ShareIcon,
    TimelineIcon,
    FoldersIcon,
  },

  mixins: [UserConfig],

  computed: {
    list(): {
      text: string;
      path: string[];
      idx: number;
    }[] {
      let path: string[] | string = this.$route.params.path || '';
      if (typeof path === 'string') {
        path = path.split('/');
      }

      return path
        .filter(Boolean) // non-empty
        .map((text, idx, arr) => {
          const path = arr.slice(0, idx + 1);
          return { text, path, idx };
        });
    },

    recursive(): boolean {
      return !!this.$route.query.recursive;
    },

    rootFolderName(): string {
      return this.routeIsPublic ? this.initstate.shareTitle : this.t('memories', 'Home');
    },
  },

  methods: {
    share() {
      _m.modals.shareNodeLink(utils.getFolderRoutePath(this.config.folders_path));
    },

    toggleRecursive() {
      this.$router.replace({
        query: {
          ...this.$router.currentRoute.query,
          recursive: this.recursive ? undefined : String(1),
        },
      });
    },

    getRoute(path: string[]) {
      return {
        ...this.$route,
        params: { path },
        hash: undefined,
      };
    },
  },
});
</script>

<style lang="scss" scoped>
.top-matter {
  .breadcrumb {
    min-width: 0;
    height: unset;
    .share-name {
      margin-left: 0.75em;
    }
  }
}
</style>
