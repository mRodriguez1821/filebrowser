<template>
  <errors v-if="error" :errorCode="error.status" />
  <div class="settings-flat-view" v-else-if="!layoutStore.loading && settings !== null">
    <form @submit.prevent="save">
      <section class="settings-section">
        <h2 class="settings-section-title">{{ t("settings.userDefaults") }}</h2>
        <p class="settings-section-desc">{{ t("settings.defaultUserDescription") }}</p>

        <user-form
          :isNew="false"
          :isDefault="true"
          v-model:user="settings.defaults"
        />
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
import UserForm from "@/components/settings/UserForm.vue";
import { useLayoutStore } from "@/stores/layout";
import Errors from "@/views/Errors.vue";
import { inject, onMounted, ref } from "vue";
import { useI18n } from "vue-i18n";

const error = ref<StatusError | null>(null);
const settings = ref<ISettings | null>(null);

const $showError = inject<IToastError>("$showError")!;
const $showSuccess = inject<IToastSuccess>("$showSuccess")!;

const { t } = useI18n();
const layoutStore = useLayoutStore();

const save = async () => {
  if (settings.value === null) return false;

  try {
    await api.update(settings.value);
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
  } catch (err) {
    if (err instanceof StatusError) {
      error.value = err;
    }
  } finally {
    layoutStore.loading = false;
  }
});
</script>
