<template>
  <div class="settings-flat-view">
    <section class="settings-section">
      <h2 class="settings-section-title">{{ t("settings.security") }}</h2>
      <p class="settings-section-desc" v-if="authStore.user?.lockPassword">
        {{ t("settings.lockPassword") }}
      </p>

      <form
        v-if="!noAuth && !authStore.user?.lockPassword"
        class="password-flat-form"
        @submit.prevent="updatePassword"
      >
        <div class="form-field">
          <label class="form-label">{{ t("settings.newPassword") }}</label>
          <input
            :class="passwordClass"
            type="password"
            :placeholder="t('settings.newPassword')"
            v-model="password"
            name="password"
            autocomplete="new-password"
          />
        </div>

        <div class="form-field">
          <label class="form-label">{{ t("settings.newPasswordConfirm") }}</label>
          <input
            :class="passwordClass"
            type="password"
            :placeholder="t('settings.newPasswordConfirm')"
            v-model="passwordConf"
            name="passwordConf"
            autocomplete="new-password"
          />
        </div>

        <div class="form-field" v-if="isCurrentPasswordRequired">
          <label class="form-label">{{ t("settings.currentPassword") }}</label>
          <input
            :class="passwordClass"
            type="password"
            :placeholder="t('settings.currentPassword')"
            v-model="currentPassword"
            name="current_password"
            autocomplete="current-password"
          />
        </div>

        <div class="settings-floating-actions">
          <button
            type="submit"
            class="button"
            :disabled="!canSubmitPassword"
          >
            <i class="material-icons">save</i>
            {{ t("buttons.update") }}
          </button>
        </div>
      </form>
    </section>
  </div>
</template>

<script setup lang="ts">
import { useAuthStore } from "@/stores/auth";
import { users as api } from "@/api";
import { computed, inject, ref } from "vue";
import { useI18n } from "vue-i18n";
import { authMethod, noAuth } from "@/utils/constants";

const authStore = useAuthStore();
const { t } = useI18n();

const $showSuccess = inject<IToastSuccess>("$showSuccess")!;
const $showError = inject<IToastError>("$showError")!;

const password = ref<string>("");
const passwordConf = ref<string>("");
const currentPassword = ref<string>("");
const isCurrentPasswordRequired = ref<boolean>(authMethod === "json");

const passwordClass = computed(() => {
  const baseClass = "input input--block";
  if (password.value === "" && passwordConf.value === "") {
    return baseClass;
  }
  if (password.value === passwordConf.value) {
    return `${baseClass} input--green`;
  }
  return `${baseClass} input--red`;
});

const canSubmitPassword = computed(() => {
  if (password.value === "" || password.value !== passwordConf.value) return false;
  if (isCurrentPasswordRequired.value && currentPassword.value === "") return false;
  return true;
});

const updatePassword = async () => {
  if (!canSubmitPassword.value || authStore.user === null) {
    return;
  }

  try {
    const data = {
      ...authStore.user,
      id: authStore.user.id,
      password: password.value,
    };
    await api.update(data, ["password"], currentPassword.value);
    authStore.updateUser(data);
    $showSuccess(t("settings.passwordUpdated"));
  } catch (e: any) {
    $showError(e);
  } finally {
    password.value = passwordConf.value = currentPassword.value = "";
  }
};
</script>
