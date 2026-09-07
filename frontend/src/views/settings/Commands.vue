<template>
  <errors v-if="error" :errorCode="error.status" />
  <div class="settings-flat-view" v-else-if="!layoutStore.loading && settings !== null">
    <form @submit.prevent="save">
      <section class="settings-section">
        <h2 class="settings-section-title">{{ t("settings.commandRunner") }}</h2>
        <i18n-t
          keypath="settings.commandRunnerHelp"
          tag="p"
          class="settings-section-desc"
          scope="global"
        >
          <code>FILE</code>
          <code>SCOPE</code>
          <a
            class="link"
            target="_blank"
            href="https://github.com/filebrowser/filebrowser/blob/master/docs/command-execution.md#hook-runner"
            >{{ t("settings.documentation") }}</a
          >
        </i18n-t>

        <div
          v-for="(command, key) in settings.commands"
          :key="key"
          class="collapsible"
        >
          <input :id="key" type="checkbox" />
          <label :for="key">
            <p>{{ capitalize(key) }}</p>
            <i class="material-icons">arrow_drop_down</i>
          </label>
          <div class="collapse">
            <textarea
              class="input input--block input--textarea"
              v-model.trim="commandObject[key]"
            ></textarea>
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
import { useLayoutStore } from "@/stores/layout";
import Errors from "@/views/Errors.vue";
import { inject, onMounted, ref } from "vue";
import { useI18n } from "vue-i18n";

const error = ref<StatusError | null>(null);
const settings = ref<ISettings | null>(null);
const commandObject = ref<{
  [key: string]: string[] | string;
}>({});

const $showError = inject<IToastError>("$showError")!;
const $showSuccess = inject<IToastSuccess>("$showSuccess")!;

const { t } = useI18n();
const layoutStore = useLayoutStore();

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

  const newSettings: ISettings = {
    ...settings.value,
    commands: {},
  };

  const keys = Object.keys(settings.value.commands) as Array<
    keyof SettingsCommand
  >;
  for (const key of keys) {
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

  try {
    await api.update(newSettings);
    $showSuccess(t("settings.settingsUpdated"));
  } catch (e: any) {
    $showError(e);
  }

  return true;
};

onMounted(async () => {
  layoutStore.loading = true;

  try {
    settings.value = await api.get();
    if (settings.value) {
      const keys = Object.keys(settings.value.commands) as Array<
        keyof SettingsCommand
      >;
      for (const key of keys) {
        const val = settings.value.commands[key];
        commandObject.value[key] = Array.isArray(val) ? val.join("\n") : (val || "");
      }
    }
  } catch (err) {
    if (err instanceof StatusError) {
      error.value = err;
    }
  } finally {
    layoutStore.loading = false;
  }
});
</script>
