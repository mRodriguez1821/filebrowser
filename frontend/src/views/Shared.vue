<template>
  <div>
    <header-bar showMenu showLogo>
      <search />
    </header-bar>

    <div class="breadcrumbs-drive-bar">
      <div class="breadcrumbs-path">
        <h1 class="breadcrumb-heading">{{ t("sidebar.sharedResources") }}</h1>
      </div>
    </div>

    <div v-if="loading">
      <h2 class="message delayed">
        <div class="spinner">
          <div class="bounce1"></div>
          <div class="bounce2"></div>
          <div class="bounce3"></div>
        </div>
        <span>{{ t("files.loading") }}</span>
      </h2>
    </div>

    <div class="settings-flat-view" v-else>
      <div class="table-responsive" v-if="links.length > 0">
        <table class="settings-table">
          <thead>
            <tr>
              <th>{{ t("settings.path") }}</th>
              <th>{{ t("settings.shareDuration") }}</th>
              <th class="action-cell"></th>
              <th class="action-cell"></th>
              <th class="action-cell"></th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="link in links" :key="link.hash">
              <td>
                <a :href="buildLink(link)" target="_blank" class="share-link-text">
                  <i class="material-icons share-file-icon">folder_shared</i>
                  <span>{{ link.path }}</span>
                </a>
              </td>
              <td>
                <template v-if="link.expire !== 0">{{
                  humanTime(link.expire)
                }}</template>
                <template v-else>{{ t("permanent") }}</template>
              </td>
              <!-- 1. Open in new tab -->
              <td class="small action-cell">
                <a
                  :href="buildLink(link)"
                  target="_blank"
                  class="table-action-btn"
                  :title="t('buttons.open')"
                  :aria-label="t('buttons.open')"
                >
                  <i class="material-icons">open_in_new</i>
                </a>
              </td>
              <!-- 2. Copy Link -->
              <td class="small action-cell">
                <button
                  class="table-action-btn copy-clipboard"
                  :aria-label="t('buttons.copyToClipboard')"
                  :title="t('buttons.copyToClipboard')"
                  @click="copyToClipboard(buildLink(link))"
                >
                  <i class="material-icons">content_paste</i>
                </button>
              </td>
              <!-- 3. Delete Share -->
              <td class="small action-cell">
                <button
                  class="table-action-btn"
                  @click="deleteLink($event, link)"
                  :aria-label="t('buttons.delete')"
                  :title="t('buttons.delete')"
                >
                  <i class="material-icons">delete</i>
                </button>
              </td>
            </tr>
          </tbody>
        </table>
      </div>

      <!-- Empty State -->
      <div class="empty-state-message" v-else>
        <i class="material-icons">folder_shared</i>
        <span>{{ t("files.lonely") }}</span>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { useAuthStore } from "@/stores/auth";
import { useLayoutStore } from "@/stores/layout";
import { share as api } from "@/api";
import dayjs from "dayjs";
import HeaderBar from "@/components/header/HeaderBar.vue";
import Search from "@/components/Search.vue";
import { inject, onMounted, ref } from "vue";
import { useI18n } from "vue-i18n";
import { copy } from "@/utils/clipboard";

const $showError = inject<IToastError>("$showError")!;
const $showSuccess = inject<IToastSuccess>("$showSuccess")!;
const { t } = useI18n();

const layoutStore = useLayoutStore();
const authStore = useAuthStore();
const loading = ref<boolean>(false);
const links = ref<Share[]>([]);

const loadShares = async () => {
  loading.value = true;
  try {
    const allLinks = await api.list();
    if (authStore.user?.perm.admin) {
      // Filter for current user only
      links.value = allLinks.filter(
        (l) => !l.userID || l.userID === authStore.user?.id
      );
    } else {
      links.value = allLinks;
    }
  } catch (err) {
    if (err instanceof Error) {
      $showError(err);
    }
  } finally {
    loading.value = false;
  }
};

onMounted(() => {
  loadShares();
});

const copyToClipboard = (text: string) => {
  copy({ text }).then(
    () => {
      $showSuccess(t("success.linkCopied"));
    },
    () => {
      copy({ text }, { permission: true }).then(
        () => {
          $showSuccess(t("success.linkCopied"));
        },
        (e) => {
          $showError(e);
        }
      );
    }
  );
};

const deleteLink = async (event: Event, link: Share) => {
  event.preventDefault();

  layoutStore.showHover({
    prompt: "share-delete",
    confirm: () => {
      layoutStore.closeHovers();
      try {
        api.remove(link.hash);
        links.value = links.value.filter((item) => item.hash !== link.hash);
        $showSuccess(t("settings.shareDeleted"));
      } catch (err) {
        if (err instanceof Error) {
          $showError(err);
        }
      }
    },
  });
};

const humanTime = (time: number) => {
  return dayjs(time * 1000).fromNow();
};

const buildLink = (share: Share) => {
  return api.getShareURL(share);
};
</script>

<style scoped>
.share-file-icon {
  font-size: 1.25em;
  vertical-align: middle;
  margin-right: 0.5em;
  color: var(--blue);
}
</style>
