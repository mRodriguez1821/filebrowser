<template>
  <div class="breadcrumbs-drive-bar">
    <!-- Home or Recent View Title -->
    <div class="breadcrumbs-path" v-if="isHomeView || isRecentView">
      <h1 class="breadcrumb-heading">{{ t("files.recentFiles") }}</h1>
    </div>

    <!-- Regular Files Explorer Breadcrumbs with Dropdown Menu -->
    <div class="breadcrumbs-path" v-else>
      <div class="breadcrumb-item-wrapper" v-if="items.length === 0">
        <button
          type="button"
          class="breadcrumb-root is-current"
          @click.stop="toggleFolderMenu"
          aria-haspopup="true"
          :aria-expanded="showFolderMenu"
        >
          <span>{{ t("sidebar.myUnit") }}</span>
          <i class="material-icons dropdown-icon">arrow_drop_down</i>
        </button>

        <!-- Dropdown Menu on Root / My Drive -->
        <div v-if="showFolderMenu" class="folder-menu-dropdown" @click.stop>
          <button
            type="button"
            class="folder-menu-item"
            @click="triggerNewFolder"
            v-if="user?.perm.create"
          >
            <i class="material-icons">create_new_folder</i>
            <span>{{ t("sidebar.newFolder") }}</span>
          </button>
          <button
            type="button"
            class="folder-menu-item"
            @click="triggerDownload"
            v-if="user?.perm.download"
          >
            <i class="material-icons">file_download</i>
            <span>{{ t("buttons.download") }}</span>
          </button>
          <button
            type="button"
            class="folder-menu-item"
            @click="triggerInfo"
          >
            <i class="material-icons">info_outline</i>
            <span>{{ t("buttons.info") }}</span>
          </button>
        </div>
      </div>

      <template v-else>
        <router-link to="/files/" class="breadcrumb-root">
          <span>{{ t("sidebar.myUnit") }}</span>
        </router-link>

        <span
          v-for="(link, index) in items"
          :key="index"
          class="breadcrumb-segment"
        >
          <i class="material-icons chevron-icon">keyboard_arrow_right</i>

          <div
            v-if="index === items.length - 1"
            class="breadcrumb-item-wrapper"
          >
            <button
              type="button"
              class="breadcrumb-link is-current"
              @click.stop="toggleFolderMenu"
              aria-haspopup="true"
              :aria-expanded="showFolderMenu"
            >
              <span>{{ link.name }}</span>
              <i class="material-icons dropdown-icon">arrow_drop_down</i>
            </button>

            <!-- Dropdown Menu on Active Folder -->
            <div v-if="showFolderMenu" class="folder-menu-dropdown" @click.stop>
              <button
                type="button"
                class="folder-menu-item"
                @click="triggerNewFolder"
                v-if="user?.perm.create"
              >
                <i class="material-icons">create_new_folder</i>
                <span>{{ t("sidebar.newFolder") }}</span>
              </button>
              <button
                type="button"
                class="folder-menu-item"
                @click="triggerDownload"
                v-if="user?.perm.download"
              >
                <i class="material-icons">file_download</i>
                <span>{{ t("buttons.download") }}</span>
              </button>
              <button
                type="button"
                class="folder-menu-item"
                @click="triggerRename"
                v-if="user?.perm.rename"
              >
                <i class="material-icons">mode_edit</i>
                <span>{{ t("buttons.rename") }}</span>
              </button>
              <button
                type="button"
                class="folder-menu-item"
                @click="triggerShare"
                v-if="user?.perm.share"
              >
                <i class="material-icons">person_add</i>
                <span>{{ t("buttons.share") }}</span>
              </button>
              <button
                type="button"
                class="folder-menu-item"
                @click="triggerInfo"
              >
                <i class="material-icons">info_outline</i>
                <span>{{ t("buttons.info") }}</span>
              </button>
              <div
                class="folder-menu-divider"
                v-if="user?.perm.delete"
              ></div>
              <button
                type="button"
                class="folder-menu-item delete-item"
                @click="triggerDelete"
                v-if="user?.perm.delete"
              >
                <i class="material-icons">delete</i>
                <span>{{ t("buttons.delete") }}</span>
              </button>
            </div>
          </div>

          <router-link
            v-else
            :to="link.url"
            class="breadcrumb-link"
          >
            <span>{{ link.name }}</span>
          </router-link>
        </span>
      </template>
    </div>

    <!-- Google Drive style right actions: segmented view toggle [ ≡ | ⊞ ] + Info button -->
    <div class="breadcrumbs-actions" v-if="authStore.isLoggedIn">
      <div class="view-mode-pill">
        <button
          type="button"
          class="view-pill-btn"
          :class="{ active: currentViewMode === 'list' }"
          @click="setViewMode('list')"
          :title="t('buttons.switchView')"
          aria-label="List view"
        >
          <i class="material-icons">view_list</i>
        </button>
        <button
          type="button"
          class="view-pill-btn"
          :class="{
            active:
              currentViewMode === 'mosaic' ||
              currentViewMode === 'mosaic gallery',
          }"
          @click="setViewMode('mosaic')"
          :title="t('buttons.switchView')"
          aria-label="Grid view"
        >
          <i class="material-icons">grid_view</i>
        </button>
      </div>

      <button
        type="button"
        class="info-icon-btn"
        @click="showInfo"
        :title="t('buttons.info')"
        aria-label="Info"
      >
        <i class="material-icons">info_outline</i>
      </button>
    </div>

    <!-- Click outside overlay to close dropdown -->
    <div
      v-if="showFolderMenu"
      class="folder-menu-overlay"
      @click="closeFolderMenu"
    />
  </div>
</template>

<script setup lang="ts">
import { computed, ref } from "vue";
import { useI18n } from "vue-i18n";
import { useRoute } from "vue-router";
import { useAuthStore } from "@/stores/auth";
import { useLayoutStore } from "@/stores/layout";
import { users, files as api } from "@/api";

const { t } = useI18n();
const route = useRoute();
const authStore = useAuthStore();
const layoutStore = useLayoutStore();

const props = defineProps<{
  base: string;
  noLink?: boolean;
}>();

const user = computed(() => authStore.user);
const showFolderMenu = ref(false);

const toggleFolderMenu = () => {
  showFolderMenu.value = !showFolderMenu.value;
};

const closeFolderMenu = () => {
  showFolderMenu.value = false;
};

const isHomeView = computed(
  () =>
    (route.path === "/files" || route.path === "/files/") &&
    route.query.view === "home"
);

const isRecentView = computed(
  () => route.path.startsWith("/files") && route.query.view === "recent"
);

const currentViewMode = computed(() => authStore.user?.viewMode || "mosaic");

const setViewMode = async (mode: ViewModeType) => {
  if (authStore.user?.viewMode === mode) return;
  const data = {
    id: authStore.user?.id,
    viewMode: mode,
  };
  try {
    if (authStore.user?.id) {
      await users.update(data, ["viewMode"]);
    }
  } catch (err) {
    console.error(err);
  }
  authStore.updateUser(data);
};

const showInfo = () => {
  layoutStore.showHover("info");
};

const triggerInfo = () => {
  closeFolderMenu();
  layoutStore.showHover("info");
};

const triggerNewFolder = () => {
  closeFolderMenu();
  layoutStore.showHover("newDir");
};

const triggerDownload = () => {
  closeFolderMenu();
  api.download("zip", route.path);
};

const triggerRename = () => {
  closeFolderMenu();
  layoutStore.showHover("rename");
};

const triggerShare = () => {
  closeFolderMenu();
  layoutStore.showHover("share");
};

const triggerDelete = () => {
  closeFolderMenu();
  layoutStore.showHover("delete");
};

const items = computed(() => {
  const relativePath = route.path.replace(props.base, "");
  const parts = relativePath.split("/");

  if (parts[0] === "") {
    parts.shift();
  }

  if (parts[parts.length - 1] === "") {
    parts.pop();
  }

  const breadcrumbs: BreadCrumb[] = [];

  for (let i = 0; i < parts.length; i++) {
    if (i === 0) {
      breadcrumbs.push({
        name: decodeURIComponent(parts[i]),
        url: props.base + "/" + parts[i] + "/",
      });
    } else {
      breadcrumbs.push({
        name: decodeURIComponent(parts[i]),
        url: breadcrumbs[i - 1].url + parts[i] + "/",
      });
    }
  }

  if (breadcrumbs.length > 3) {
    while (breadcrumbs.length !== 4) {
      breadcrumbs.shift();
    }

    breadcrumbs[0].name = "...";
  }

  return breadcrumbs;
});
</script>
