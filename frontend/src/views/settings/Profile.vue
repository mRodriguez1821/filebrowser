<template>
  <div class="settings-flat-view">
    <!-- 1. Almacenamiento -->
    <section class="settings-section">
      <h2 class="settings-section-title">{{ t("settings.storage") }}</h2>
      <div class="storage-info-box">
        <p class="storage-usage-label">
          {{
            t("sidebar.storageUsed", {
              used: usage.used,
              total: usage.total,
            })
          }}
        </p>
        <div class="storage-bar-track">
          <div
            class="storage-bar-fill"
            :style="{ width: `${usage.usedPercentage}%` }"
          ></div>
        </div>
      </div>
    </section>

    <div class="settings-divider"></div>

    <!-- 2. Página de inicio -->
    <section class="settings-section">
      <h2 class="settings-section-title">{{ t("settings.startPage") }}</h2>
      <div class="radio-group">
        <label class="radio-label">
          <input
            type="radio"
            name="startPage"
            value="home"
            v-model="startPage"
            @change="handleStartPageChange"
          />
          <span class="radio-custom"></span>
          <span class="radio-text">{{ t("settings.startPageHome") }}</span>
        </label>
        <label class="radio-label">
          <input
            type="radio"
            name="startPage"
            value="myUnit"
            v-model="startPage"
            @change="handleStartPageChange"
          />
          <span class="radio-custom"></span>
          <span class="radio-text">{{ t("settings.startPageUnit") }}</span>
        </label>
      </div>
    </section>

    <div class="settings-divider"></div>

    <!-- 3. Apariencia -->
    <section class="settings-section">
      <h2 class="settings-section-title">{{ t("settings.appearance") }}</h2>
      <div class="radio-group">
        <label class="radio-label">
          <input
            type="radio"
            name="theme"
            value="light"
            v-model="selectedTheme"
            @change="handleThemeChange('light')"
          />
          <span class="radio-custom"></span>
          <span class="radio-text">{{ t("settings.themeLight") }}</span>
        </label>
        <label class="radio-label">
          <input
            type="radio"
            name="theme"
            value="dark"
            v-model="selectedTheme"
            @change="handleThemeChange('dark')"
          />
          <span class="radio-custom"></span>
          <span class="radio-text">{{ t("settings.themeDark") }}</span>
        </label>
        <label class="radio-label">
          <input
            type="radio"
            name="theme"
            value=""
            v-model="selectedTheme"
            @change="handleThemeChange('')"
          />
          <span class="radio-custom"></span>
          <span class="radio-text">{{ t("settings.themeSystem") }}</span>
        </label>
      </div>
    </section>

    <div class="settings-divider"></div>

    <!-- 4. Preferencias de Archivos y Editor -->
    <section class="settings-section">
      <h2 class="settings-section-title">{{ t("settings.filePreferences") }}</h2>
      <div class="checkbox-group">
        <label class="checkbox-label">
          <input type="checkbox" v-model="hideDotfiles" />
          <span>{{ t("settings.hideDotfiles") }}</span>
        </label>
        <label class="checkbox-label">
          <input type="checkbox" v-model="singleClick" />
          <span>{{ t("settings.singleClick") }}</span>
        </label>
        <label class="checkbox-label">
          <input type="checkbox" v-model="redirectAfterCopyMove" />
          <span>{{ t("settings.redirectAfterCopyMove") }}</span>
        </label>
        <label class="checkbox-label">
          <input type="checkbox" v-model="dateFormat" />
          <span>{{ t("settings.setDateFormat") }}</span>
        </label>
      </div>

      <div class="form-row-group">
        <div class="form-field">
          <label class="form-label">{{ t("settings.language") }}</label>
          <languages class="input input--block" v-model:locale="locale" />
        </div>

        <div class="form-field">
          <label class="form-label">{{ t("settings.aceEditorTheme") }}</label>
          <AceEditorTheme
            class="input input--block"
            v-model:aceEditorTheme="aceEditorTheme"
            id="aceTheme"
          />
        </div>
      </div>

      <div class="settings-floating-actions">
        <button
          type="button"
          class="button"
          @click="updateSettings"
        >
          <i class="material-icons">save</i>
          {{ t("buttons.update") }}
        </button>
      </div>
    </section>
  </div>
</template>

<script setup lang="ts">
import { useAuthStore } from "@/stores/auth";
import { useLayoutStore } from "@/stores/layout";
import { users as api, files as filesApi } from "@/api";
import AceEditorTheme from "@/components/settings/AceEditorTheme.vue";
import Languages from "@/components/settings/Languages.vue";
import { setTheme } from "@/utils/theme";
import { inject, onMounted, reactive, ref } from "vue";
import { useI18n } from "vue-i18n";
import prettyBytes from "pretty-bytes";

const layoutStore = useLayoutStore();
const authStore = useAuthStore();
const { t } = useI18n();

const $showSuccess = inject<IToastSuccess>("$showSuccess")!;
const $showError = inject<IToastError>("$showError")!;

const hideDotfiles = ref<boolean>(false);
const singleClick = ref<boolean>(false);
const redirectAfterCopyMove = ref<boolean>(false);
const dateFormat = ref<boolean>(false);
const locale = ref<string>("");
const aceEditorTheme = ref<string>("");

// Storage & Start page & Theme
const usage = reactive({ used: "0 B", total: "0 B", usedPercentage: 0 });
const startPage = ref<string>(localStorage.getItem("fb_start_page") || "home");
const selectedTheme = ref<UserTheme>("");

const handleStartPageChange = () => {
  localStorage.setItem("fb_start_page", startPage.value);
  $showSuccess(t("settings.settingsUpdated"));
};

const handleThemeChange = (newTheme: UserTheme) => {
  selectedTheme.value = newTheme;
  setTheme(newTheme);
  if (authStore.user) {
    const data = { ...authStore.user, id: authStore.user.id, theme: newTheme };
    api.update(data, ["theme"]).catch(() => {});
    authStore.updateUser(data);
  }
};

onMounted(async () => {
  layoutStore.loading = true;
  if (authStore.user === null) {
    layoutStore.loading = false;
    return false;
  }

  locale.value = authStore.user.locale;
  hideDotfiles.value = authStore.user.hideDotfiles;
  singleClick.value = authStore.user.singleClick;
  redirectAfterCopyMove.value = authStore.user.redirectAfterCopyMove;
  dateFormat.value = authStore.user.dateFormat;
  aceEditorTheme.value = authStore.user.aceEditorTheme;
  selectedTheme.value = (document.documentElement.className as UserTheme) || "";

  try {
    const abort = new AbortController();
    const res = await filesApi.usage("/", abort.signal);
    usage.used = prettyBytes(res.used, { binary: true });
    usage.total = prettyBytes(res.total, { binary: true });
    usage.usedPercentage = Math.round((res.used / res.total) * 100);
  } catch {
    // ignore
  }

  layoutStore.loading = false;
  return true;
});

const updateSettings = async (event?: Event) => {
  if (event) event.preventDefault();

  try {
    if (authStore.user === null) throw new Error("User is not set!");

    const data = {
      ...authStore.user,
      id: authStore.user.id,
      locale: locale.value,
      hideDotfiles: hideDotfiles.value,
      singleClick: singleClick.value,
      redirectAfterCopyMove: redirectAfterCopyMove.value,
      dateFormat: dateFormat.value,
      aceEditorTheme: aceEditorTheme.value,
    };

    await api.update(data, [
      "locale",
      "hideDotfiles",
      "singleClick",
      "redirectAfterCopyMove",
      "dateFormat",
      "aceEditorTheme",
    ]);
    authStore.updateUser(data);
    $showSuccess(t("settings.settingsUpdated"));
  } catch (err) {
    if (err instanceof Error) {
      $showError(err);
    }
  }
};
</script>
