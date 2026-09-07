<template>
  <div v-show="active" @click="closeHovers" class="overlay"></div>
  <nav :class="{ active }">
    <!-- SETTINGS SIDEBAR -->
    <template v-if="isSettings">
      <div class="nav-links-section" v-if="isLoggedIn">
        <button
          type="button"
          class="nav-pill-item"
          :class="{ active: isGeneralActive }"
          @click="toGeneral"
        >
          <i class="material-icons">tune</i>
          <span>{{ t("settings.general") }}</span>
        </button>

        <button
          type="button"
          class="nav-pill-item"
          :class="{ active: isSecurityActive }"
          @click="toSecurity"
          v-if="!noAuth && !user?.lockPassword"
        >
          <i class="material-icons">lock</i>
          <span>{{ t("settings.security") }}</span>
        </button>

        <button
          type="button"
          class="nav-pill-item"
          :class="{ active: isSharesSettingsActive }"
          @click="toSharesSettings"
          v-if="user?.perm.share"
        >
          <i class="material-icons">people</i>
          <span>{{ t("sidebar.sharedResources") }}</span>
        </button>

        <button
          type="button"
          class="nav-pill-item"
          :class="{ active: isGlobalActive }"
          @click="toGlobal"
          v-if="user?.perm.admin"
        >
          <i class="material-icons">public</i>
          <span>{{ t("settings.globalSettings") }}</span>
        </button>

        <button
          type="button"
          class="nav-pill-item"
          :class="{ active: isDefaultsActive }"
          @click="toDefaults"
          v-if="user?.perm.admin"
        >
          <i class="material-icons">person_outline</i>
          <span>{{ t("settings.userDefaults") }}</span>
        </button>

        <button
          type="button"
          class="nav-pill-item"
          :class="{ active: isCommandsActive }"
          @click="toCommands"
          v-if="user?.perm.admin && enableExec"
        >
          <i class="material-icons">terminal</i>
          <span>{{ t("settings.commandRunner") }}</span>
        </button>

        <button
          type="button"
          class="nav-pill-item"
          :class="{ active: isUsersActive }"
          @click="toUsers"
          v-if="user?.perm.admin"
        >
          <i class="material-icons">manage_accounts</i>
          <span>{{ t("settings.users") }}</span>
        </button>
      </div>

      <!-- Spacer -->
      <div class="nav-spacer"></div>

      <!-- Red Logout Button -->
      <div class="logout-section" v-if="canLogout && isLoggedIn">
        <button
          type="button"
          class="btn-logout-pill"
          @click="handleLogout"
          :title="t('sidebar.logout')"
          :aria-label="t('sidebar.logout')"
        >
          <i class="material-icons">logout</i>
          <span>{{ t("sidebar.logout") }}</span>
        </button>
      </div>
    </template>

    <!-- FILES EXPLORER SIDEBAR -->
    <template v-else>
      <!-- Top: + Nuevo Button (Google Drive Style) -->
      <div class="new-action-wrapper sidebar-new-action" v-if="isLoggedIn && user?.perm.create">
        <button
          type="button"
          class="btn-new"
          @click="toggleNewMenu"
          :aria-expanded="showNewMenu"
          :aria-label="t('sidebar.new')"
        >
          <i class="material-icons">add</i>
          <span>{{ t("sidebar.new") }}</span>
        </button>

        <!-- Dropdown for + Nuevo -->
        <div v-if="showNewMenu" class="new-menu-dropdown">
          <button type="button" class="new-menu-item" @click="triggerNewFolder">
            <i class="material-icons">create_new_folder</i>
            <span>{{ t("sidebar.newFolder") }}</span>
          </button>
          <button type="button" class="new-menu-item" @click="triggerNewFile">
            <i class="material-icons">note_add</i>
            <span>{{ t("sidebar.newFile") }}</span>
          </button>
          <div class="new-menu-divider"></div>
          <button type="button" class="new-menu-item" @click="triggerUploadFile">
            <i class="material-icons">upload_file</i>
            <span>{{ t("sidebar.uploadFile") }}</span>
          </button>
          <button
            type="button"
            class="new-menu-item"
            @click="triggerUploadFolder"
          >
            <i class="material-icons">drive_folder_upload</i>
            <span>{{ t("sidebar.uploadFolder") }}</span>
          </button>
        </div>
      </div>

      <!-- Navigation Links -->
      <div class="nav-links-section" v-if="isLoggedIn">
        <button
          type="button"
          class="nav-pill-item"
          :class="{ active: isHomeActive }"
          @click="toHome"
        >
          <i class="material-icons">home</i>
          <span>{{ t("sidebar.home") }}</span>
        </button>

        <button
          type="button"
          class="nav-pill-item"
          :class="{ active: isMyDriveActive }"
          @click="toMyDrive"
        >
          <i class="material-icons">folder</i>
          <span>{{ t("sidebar.myUnit") }}</span>
        </button>

        <button
          type="button"
          class="nav-pill-item"
          :class="{ active: isSharesActive }"
          @click="toShares"
        >
          <i class="material-icons">people</i>
          <span>{{ t("sidebar.sharedResources") }}</span>
        </button>

        <button
          type="button"
          class="nav-pill-item"
          :class="{ active: isRecentActive }"
          @click="toRecent"
        >
          <i class="material-icons">schedule</i>
          <span>{{ t("sidebar.recent") }}</span>
        </button>
      </div>

      <!-- Non-logged in navigation -->
      <div class="nav-links-section" v-else>
        <router-link
          v-if="!hideLoginButton"
          class="nav-pill-item"
          to="/login"
          :aria-label="t('sidebar.login')"
        >
          <i class="material-icons">exit_to_app</i>
          <span>{{ t("sidebar.login") }}</span>
        </router-link>

        <router-link
          v-if="signup"
          class="nav-pill-item"
          to="/login"
          :aria-label="t('sidebar.signup')"
        >
          <i class="material-icons">person_add</i>
          <span>{{ t("sidebar.signup") }}</span>
        </router-link>
      </div>

      <!-- Spacer -->
      <div class="nav-spacer"></div>

      <!-- Storage Info Widget -->
      <div
        class="storage-section"
        v-if="isFiles && !disableUsedPercentage && isLoggedIn"
      >
        <div class="storage-header">
          <i class="material-icons">cloud_queue</i>
          <span>{{ t("sidebar.storage") }}</span>
        </div>
        <div class="storage-bar-track">
          <div
            class="storage-bar-fill"
            :style="{ width: `${usage.usedPercentage}%` }"
          ></div>
        </div>
        <div class="storage-text">
          {{
            t("sidebar.storageUsed", {
              used: usage.used,
              total: usage.total,
            })
          }}
        </div>
      </div>

      <!-- Red Logout Button -->
      <div class="logout-section" v-if="canLogout && isLoggedIn">
        <button
          type="button"
          class="btn-logout-pill"
          @click="handleLogout"
          :title="t('sidebar.logout')"
          :aria-label="t('sidebar.logout')"
        >
          <i class="material-icons">logout</i>
          <span>{{ t("sidebar.logout") }}</span>
        </button>
      </div>
    </template>
  </nav>

  <!-- Mobile / Tablet Floating Action Button (+ Nuevo) -->
  <div
    class="mobile-fab-wrapper"
    v-if="!isSettings && isLoggedIn && user?.perm.create"
  >
    <button
      type="button"
      class="btn-fab-new"
      @click="toggleFabMenu"
      :aria-expanded="showFabMenu"
      :aria-label="t('sidebar.new')"
      :title="t('sidebar.new')"
    >
      <i class="material-icons">add</i>
    </button>

    <!-- Dropdown for Mobile FAB (opens upward) -->
    <div v-if="showFabMenu" class="fab-menu-dropdown">
      <button type="button" class="new-menu-item" @click="triggerNewFolder">
        <i class="material-icons">create_new_folder</i>
        <span>{{ t("sidebar.newFolder") }}</span>
      </button>
      <button type="button" class="new-menu-item" @click="triggerNewFile">
        <i class="material-icons">note_add</i>
        <span>{{ t("sidebar.newFile") }}</span>
      </button>
      <div class="new-menu-divider"></div>
      <button type="button" class="new-menu-item" @click="triggerUploadFile">
        <i class="material-icons">upload_file</i>
        <span>{{ t("sidebar.uploadFile") }}</span>
      </button>
      <button
        type="button"
        class="new-menu-item"
        @click="triggerUploadFolder"
      >
        <i class="material-icons">drive_folder_upload</i>
        <span>{{ t("sidebar.uploadFolder") }}</span>
      </button>
    </div>

    <!-- Backdrop overlay when FAB dropdown is open -->
    <div
      v-if="showFabMenu"
      class="fab-backdrop"
      @click="closeFabMenu"
    />
  </div>
</template>

<script setup lang="ts">
import { computed, onBeforeUnmount, reactive, ref, watch } from "vue";
import { useAuthStore } from "@/stores/auth";
import { useFileStore } from "@/stores/file";
import { useLayoutStore } from "@/stores/layout";
import { useI18n } from "vue-i18n";
import { useRoute, useRouter } from "vue-router";
import prettyBytes from "pretty-bytes";
import * as auth from "@/utils/auth";
import * as upload from "@/utils/upload";
import buttons from "@/utils/buttons";
import { files as api } from "@/api";
import {
  disableUsedPercentage,
  enableExec,
  hideLoginButton,
  loginPage,
  logoutPage,
  noAuth,
  signup,
} from "@/utils/constants";

const authStore = useAuthStore();
const fileStore = useFileStore();
const layoutStore = useLayoutStore();
const { t } = useI18n();
const route = useRoute();
const router = useRouter();

const showNewMenu = ref(false);
const showFabMenu = ref(false);

const toggleFabMenu = () => {
  showFabMenu.value = !showFabMenu.value;
};

const closeFabMenu = () => {
  showFabMenu.value = false;
};

const USAGE_DEFAULT = { used: "0 B", total: "0 B", usedPercentage: 0 };
const usage = reactive({ ...USAGE_DEFAULT });
let usageAbortController = new AbortController();

const active = computed(() => layoutStore.currentPromptName === "sidebar");
const isLoggedIn = computed(() => authStore.isLoggedIn);
const user = computed(() => authStore.user);
const isFiles = computed(() => fileStore.isFiles);
const canLogout = computed(
  () => !noAuth && (loginPage || logoutPage !== "/login")
);

// Settings routes
const isSettings = computed(() => route.path.startsWith("/settings"));

const isGeneralActive = computed(
  () =>
    route.path === "/settings/profile" ||
    route.path === "/settings" ||
    route.path === "/settings/"
);

const isSecurityActive = computed(
  () => route.path === "/settings/security"
);

const isSharesSettingsActive = computed(
  () => route.path.startsWith("/settings/shares")
);

const isGlobalActive = computed(() =>
  route.path.startsWith("/settings/global")
);

const isDefaultsActive = computed(() =>
  route.path.startsWith("/settings/defaults")
);

const isCommandsActive = computed(() =>
  route.path.startsWith("/settings/commands")
);

const isUsersActive = computed(
  () => route.path.startsWith("/settings/users") || route.name === "User"
);

// Route active states
const isHomeActive = computed(
  () =>
    (route.path === "/files" || route.path === "/files/") &&
    route.query.view === "home"
);

const isMyDriveActive = computed(
  () => route.path.startsWith("/files") && !route.query.view
);

const isSharesActive = computed(
  () => route.path === "/shared" || route.path === "/shared/"
);

const isRecentActive = computed(
  () => route.path.startsWith("/files") && route.query.view === "recent"
);

const toGeneral = () => {
  router.push({ path: "/settings/profile" });
  closeHovers();
};

const toSecurity = () => {
  router.push({ path: "/settings/security" });
  closeHovers();
};

const toSharesSettings = () => {
  router.push({ path: "/settings/shares" });
  closeHovers();
};

const toGlobal = () => {
  router.push({ path: "/settings/global" });
  closeHovers();
};

const toDefaults = () => {
  router.push({ path: "/settings/defaults" });
  closeHovers();
};

const toCommands = () => {
  router.push({ path: "/settings/commands" });
  closeHovers();
};

const toUsers = () => {
  router.push({ path: "/settings/users" });
  closeHovers();
};

const toggleNewMenu = () => {
  showNewMenu.value = !showNewMenu.value;
};

const closeHovers = () => {
  layoutStore.closeHovers();
  showNewMenu.value = false;
  showFabMenu.value = false;
};

const triggerNewFolder = () => {
  showNewMenu.value = false;
  showFabMenu.value = false;
  layoutStore.showHover("newDir");
};

const triggerNewFile = () => {
  showNewMenu.value = false;
  showFabMenu.value = false;
  layoutStore.showHover("newFile");
};

const openUploadPicker = (isFolder: boolean) => {
  showNewMenu.value = false;
  showFabMenu.value = false;
  const input = document.createElement("input");
  input.type = "file";
  input.multiple = true;
  if (isFolder) {
    (input as any).webkitdirectory = true;
  }
  input.onchange = async (event: Event) => {
    const files = (event.currentTarget as HTMLInputElement)?.files;
    if (!files || files.length === 0) return;

    const folder_upload = !!files[0].webkitRelativePath;
    const uploadFiles: UploadList = [];
    for (let i = 0; i < files.length; i++) {
      const file = files[i];
      const fullPath = folder_upload ? file.webkitRelativePath : undefined;
      uploadFiles.push({
        file,
        name: file.name,
        size: file.size,
        isDir: false,
        fullPath,
      });
    }

    const currentPath = route.path.endsWith("/") ? route.path : route.path + "/";
    buttons.loading("upload");
    const conflict = await upload.checkConflict(uploadFiles, currentPath);

    if (conflict.length > 0) {
      buttons.done("upload");
      layoutStore.showHover({
        prompt: "resolve-conflict",
        props: {
          conflict: conflict,
          isUploadAction: true,
        },
        confirm: (e: Event, result: Array<ConflictingResource>) => {
          e.preventDefault();
          layoutStore.closeHovers();
          for (let i = result.length - 1; i >= 0; i--) {
            const item = result[i];
            if (item.checked.length === 2) {
              continue;
            } else if (
              item.checked.length === 1 &&
              item.checked[0] === "origin"
            ) {
              uploadFiles[item.index].overwrite = true;
            } else {
              uploadFiles.splice(item.index, 1);
            }
          }
          if (uploadFiles.length > 0) {
            upload.handleFiles(uploadFiles, currentPath);
          }
        },
      });
      return;
    }

    upload.handleFiles(uploadFiles, currentPath);
  };
  input.click();
};

const triggerUploadFile = () => openUploadPicker(false);
const triggerUploadFolder = () => openUploadPicker(true);

const toHome = () => {
  router.push({ path: "/files/", query: { view: "home" } });
  closeHovers();
};

const toMyDrive = () => {
  router.push({ path: "/files/" });
  closeHovers();
};

const toShares = () => {
  router.push({ path: "/shared" });
  closeHovers();
};

const toRecent = () => {
  router.push({ path: "/files/", query: { view: "recent" } });
  closeHovers();
};

const handleLogout = () => {
  closeHovers();
  auth.logout();
};

const fetchUsage = async () => {
  if (disableUsedPercentage) return;
  const currentPath = route.path.endsWith("/") ? route.path : route.path + "/";
  try {
    usageAbortController.abort();
    usageAbortController = new AbortController();
    const res = await api.usage(currentPath, usageAbortController.signal);
    usage.used = prettyBytes(res.used, { binary: true });
    usage.total = prettyBytes(res.total, { binary: true });
    usage.usedPercentage = Math.round((res.used / res.total) * 100);
  } catch {
    // ignore fetch aborts
  }
};

watch(
  () => route.path,
  (newPath) => {
    if (newPath.includes("/files")) {
      fetchUsage();
    }
  },
  { immediate: true }
);

onBeforeUnmount(() => {
  usageAbortController.abort();
});
</script>
