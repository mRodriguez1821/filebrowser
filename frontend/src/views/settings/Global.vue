<template>
  <errors v-if="error" :errorCode="error.status" />
  <div class="settings-flat-view" v-else-if="!layoutStore.loading && settings !== null">
    <form @submit.prevent="save">
      <section class="settings-section">
        <h2 class="settings-section-title">{{ t("settings.globalSettings") }}</h2>

        <div class="checkbox-group">
          <label class="checkbox-label">
            <input type="checkbox" v-model="settings.signup" />
            <span>{{ t("settings.allowSignup") }}</span>
          </label>

          <label class="checkbox-label">
            <input type="checkbox" v-model="settings.createUserDir" />
            <span>{{ t("settings.createUserDir") }}</span>
          </label>

          <label class="checkbox-label">
            <input type="checkbox" v-model="settings.hideLoginButton" />
            <span>{{ t("settings.hideLoginButton") }}</span>
          </label>
        </div>

        <div class="form-row-group">
          <div class="form-field">
            <label class="form-label">{{ t("settings.userHomeBasePath") }}</label>
            <input
              class="input input--block"
              type="text"
              v-model="settings.userHomeBasePath"
            />
          </div>

          <div class="form-field">
            <label class="form-label" for="minimumPasswordLength">{{
              t("settings.minimumPasswordLength")
            }}</label>
            <vue-number-input
              controls
              v-model.number="settings.minimumPasswordLength"
              id="minimumPasswordLength"
              :min="1"
            />
          </div>
        </div>

        <h3>{{ t("settings.rules") }}</h3>
        <p class="small">{{ t("settings.globalRules") }}</p>
        <rules v-model:rules="settings.rules" />

        <div v-if="enableExec">
          <h3>{{ t("settings.executeOnShell") }}</h3>
          <p class="small">{{ t("settings.executeOnShellDescription") }}</p>
          <input
            class="input input--block"
            type="text"
            placeholder="bash -c, cmd /c, ..."
            v-model="shellValue"
          />
        </div>

        <h3>{{ t("settings.branding") }}</h3>

        <i18n-t
          keypath="settings.brandingHelp"
          tag="p"
          class="small"
          scope="global"
        >
          <a
            class="link"
            target="_blank"
            href="https://github.com/filebrowser/filebrowser/blob/master/docs/customization.md#custom-branding"
            >{{ t("settings.documentation") }}</a
          >
        </i18n-t>

        <div class="checkbox-group">
          <label class="checkbox-label">
            <input
              type="checkbox"
              v-model="settings.branding.disableExternal"
              id="branding-links"
            />
            <span>{{ t("settings.disableExternalLinks") }}</span>
          </label>

          <label class="checkbox-label">
            <input
              type="checkbox"
              v-model="settings.branding.disableUsedPercentage"
              id="branding-used-disk"
            />
            <span>{{ t("settings.disableUsedDiskPercentage") }}</span>
          </label>
        </div>

        <div class="form-row-group">
          <div class="form-field">
            <label class="form-label" for="theme">{{ t("settings.themes.title") }}</label>
            <themes
              class="input input--block"
              v-model:theme="settings.branding.theme"
              id="theme"
            ></themes>
          </div>

          <div class="form-field">
            <label class="form-label" for="branding-name">{{ t("settings.instanceName") }}</label>
            <input
              class="input input--block"
              type="text"
              v-model="settings.branding.name"
              id="branding-name"
            />
          </div>
        </div>

        <div class="form-field">
          <label class="form-label" for="branding-files">{{
            t("settings.brandingDirectoryPath")
          }}</label>
          <input
            class="input input--block"
            type="text"
            v-model="settings.branding.files"
            id="branding-files"
          />
        </div>

        <h3>{{ t("settings.tusUploads") }}</h3>
        <p class="small">{{ t("settings.tusUploadsHelp") }}</p>

        <div class="form-row-group">
          <div class="form-field">
            <label class="form-label" for="tus-chunkSize">{{
              t("settings.tusUploadsChunkSize")
            }}</label>
            <input
              class="input input--block"
              type="text"
              v-model="formattedChunkSize"
              id="tus-chunkSize"
            />
          </div>

          <div class="form-field">
            <label class="form-label" for="tus-retryCount">{{
              t("settings.tusUploadsRetryCount")
            }}</label>
            <vue-number-input
              controls
              v-model.number="settings.tus.retryCount"
              id="tus-retryCount"
              :min="0"
            />
          </div>
        </div>
      </section>

      <div class="settings-floating-actions">
        <button class="button" type="submit">
          <i class="material-icons">save</i>
          {{ t("buttons.update") }}
        </button>
      </div>
    </form>
  </div>
</template>

<script setup lang="ts">
import { settings as api } from "@/api";
import { StatusError } from "@/api/utils";
import Rules from "@/components/settings/Rules.vue";
import Themes from "@/components/settings/Themes.vue";
import { useLayoutStore } from "@/stores/layout";
import { enableExec } from "@/utils/constants";
import { getTheme, setTheme } from "@/utils/theme";
import Errors from "@/views/Errors.vue";
import { computed, inject, onBeforeUnmount, onMounted, ref } from "vue";
import { useI18n } from "vue-i18n";

const error = ref<StatusError | null>(null);
const originalSettings = ref<ISettings | null>(null);
const settings = ref<ISettings | null>(null);
const debounceTimeout = ref<number | null>(null);
const pendingChunkSize = ref<string | null>(null);

const commandObject = ref<{
  [key: string]: string[] | string;
}>({});
const shellValue = ref<string>("");

const $showError = inject<IToastError>("$showError")!;
const $showSuccess = inject<IToastSuccess>("$showSuccess")!;

const { t } = useI18n();

const layoutStore = useLayoutStore();

const formattedChunkSize = computed({
  get() {
    return settings?.value?.tus?.chunkSize
      ? formatBytes(settings?.value?.tus?.chunkSize)
      : "";
  },
  set(value: string) {
    // Use debouncing to allow the user to type freely without
    // interruption by the formatter
    // Clear the previous timeout if it exists
    if (debounceTimeout.value) {
      clearTimeout(debounceTimeout.value);
    }

    pendingChunkSize.value = value;

    // Set a new timeout to apply the format after a short delay
    debounceTimeout.value = window.setTimeout(applyChunkSize, 1500);
  },
});

// applyChunkSize commits what the user typed. Saving must flush it first:
// otherwise submitting within the debounce window persists the previous value,
// and the setting appears to refuse the change.
const applyChunkSize = () => {
  if (debounceTimeout.value) {
    clearTimeout(debounceTimeout.value);
    debounceTimeout.value = null;
  }

  if (pendingChunkSize.value === null) return;

  if (settings.value) {
    settings.value.tus.chunkSize = parseBytes(pendingChunkSize.value);
  }
  pendingChunkSize.value = null;
};

// Define funcs
const capitalize = (name: string, where: string | RegExp = "_") => {
  if (where === "caps") where = /(?=[A-Z])/;
  const split = name.split(where);
  name = "";

  for (let i = 0; i < split.length; i++) {
    name += split[i].charAt(0).toUpperCase() + split[i].slice(1) + " ";
  }

  return name.slice(0, -1);
};

const save = async () => {
  if (settings.value === null) return false;
  applyChunkSize();
  const newSettings: ISettings = {
    ...settings.value,
    shell:
      settings.value?.shell
        .join(" ")
        .trim()
        .split(" ")
        .filter((s: string) => s !== "") ?? [],
    commands: {},
  };

  const keys = Object.keys(settings.value.commands) as Array<
    keyof SettingsCommand
  >;
  for (const key of keys) {
    // not sure if we can safely assume non-null
    const newValue = commandObject.value[key];
    if (!newValue) continue;

    if (Array.isArray(newValue)) {
      newSettings.commands[key] = newValue;
    } else if (key in commandObject.value) {
      newSettings.commands[key] = newValue
        .split("\n")
        .filter((cmd: string) => cmd !== "");
    }
  }
  newSettings.shell = shellValue.value
    .trim()
    .split(" ")
    .filter((s) => s !== "");

  if (newSettings.branding.theme !== getTheme()) {
    setTheme(newSettings.branding.theme);
  }

  try {
    await api.update(newSettings);
    $showSuccess(t("settings.settingsUpdated"));
  } catch (e: any) {
    $showError(e);
  }

  return true;
};
// Parse the user-friendly input (e.g., "20M" or "1T") to bytes
const parseBytes = (input: string) => {
  const regex = /^(\d+)(\.\d+)?(B|K|KB|M|MB|G|GB|T|TB)?$/i;
  const matches = input.match(regex);
  if (matches) {
    const size = parseFloat(matches[1].concat(matches[2] || ""));
    // The unit is optional: a bare number is already a count of bytes. Reading
    // it unguarded throws, and the throw happens inside the debounce callback,
    // so the setting is silently never updated.
    let unit: keyof SettingsUnit = (
      matches[3] ?? "B"
    ).toUpperCase() as keyof SettingsUnit;
    if (!unit.endsWith("B")) {
      unit += "B";
    }
    const units: SettingsUnit = {
      KB: 1024,
      MB: 1024 ** 2,
      GB: 1024 ** 3,
      TB: 1024 ** 4,
    };
    return size * (units[unit as keyof SettingsUnit] || 1);
  } else {
    return 1024 ** 2;
  }
};
// Format the chunk size in bytes to user-friendly format
const formatBytes = (bytes: number) => {
  const units = ["B", "KB", "MB", "GB", "TB"];
  let size = bytes;
  let unitIndex = 0;
  while (size >= 1024 && unitIndex < units.length - 1) {
    size /= 1024;
    unitIndex++;
  }
  return `${size}${units[unitIndex]}`;
};

// Define Hooks

onMounted(async () => {
  try {
    layoutStore.loading = true;
    const original: ISettings = await api.get();
    const newSettings: ISettings = { ...original, commands: {} };

    const keys = Object.keys(original.commands) as Array<keyof SettingsCommand>;
    for (const key of keys) {
      newSettings.commands[key] = original.commands[key];
      commandObject.value[key] = original.commands[key]!.join("\n");
    }

    originalSettings.value = original;
    settings.value = newSettings;
    shellValue.value = newSettings.shell.join(" ");
  } catch (err) {
    if (err instanceof Error) {
      error.value = err;
    }
  } finally {
    layoutStore.loading = false;
  }
});

// Clear the debounce timeout when the component is destroyed
onBeforeUnmount(() => {
  if (debounceTimeout.value) {
    clearTimeout(debounceTimeout.value);
  }
});
</script>
