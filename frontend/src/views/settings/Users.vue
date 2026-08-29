<template>
  <errors v-if="error" :errorCode="error.status" />
  <div class="settings-flat-view" v-else-if="!layoutStore.loading">
    <div class="settings-header-action-row">
      <h2 class="settings-section-title">{{ t("settings.users") }}</h2>
      <router-link to="/settings/users/new">
        <button class="button">
          {{ t("buttons.new") }}
        </button>
      </router-link>
    </div>

    <div class="table-responsive">
      <table class="settings-table">
        <thead>
          <tr>
            <th>{{ t("settings.username") }}</th>
            <th>{{ t("settings.admin") }}</th>
            <th>{{ t("settings.scope") }}</th>
            <th class="action-cell"></th>
          </tr>
        </thead>
        <tbody>
          <tr v-for="user in users" :key="user.id">
            <td class="user-name-cell">{{ user.username }}</td>
            <td>
              <i v-if="user.perm.admin" class="material-icons check-icon">done</i>
              <i v-else class="material-icons close-icon">close</i>
            </td>
            <td>{{ user.scope }}</td>
            <td class="small action-cell">
              <router-link
                :to="'/settings/users/' + user.id"
                class="table-action-btn"
                :title="t('buttons.edit')"
              >
                <i class="material-icons">mode_edit</i>
              </router-link>
            </td>
          </tr>
        </tbody>
      </table>
    </div>
  </div>
</template>

<script setup lang="ts">
import { useLayoutStore } from "@/stores/layout";
import { users as api } from "@/api";
import Errors from "@/views/Errors.vue";
import { onMounted, ref } from "vue";
import { useI18n } from "vue-i18n";
import { StatusError } from "@/api/utils";

const error = ref<StatusError | null>(null);
const users = ref<IUser[]>([]);

const layoutStore = useLayoutStore();
const { t } = useI18n();

onMounted(async () => {
  layoutStore.loading = true;

  try {
    users.value = await api.getAll();
  } catch (err) {
    if (err instanceof Error) {
      error.value = err;
    }
  } finally {
    layoutStore.loading = false;
  }
});
</script>
