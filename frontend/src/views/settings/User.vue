<template>
  <errors v-if="error" :errorCode="error.status" />
  <div class="settings-flat-view" v-else-if="!layoutStore.loading">
    <form @submit="save">
      <section class="settings-section">
        <h2 class="settings-section-title" v-if="user?.id === 0">{{ $t("settings.newUser") }}</h2>
        <h2 class="settings-section-title" v-else>{{ $t("settings.user") }} {{ user?.username }}</h2>

        <div v-if="user">
          <user-form
            v-model:user="user"
            v-model:createUserDir="createUserDir"
            :isDefault="false"
            :isNew="isNew"
          />
        </div>

        <div class="settings-floating-actions">
          <button
            v-if="!isNew"
            @click.prevent="deletePrompt"
            type="button"
            class="button button--red"
            :aria-label="$t('buttons.delete')"
            :title="$t('buttons.delete')"
          >
            <i class="material-icons">delete</i>
            {{ $t("buttons.delete") }}
          </button>
          <router-link to="/settings/users">
            <button
              class="button button--grey"
              :aria-label="$t('buttons.cancel')"
              :title="$t('buttons.cancel')"
              type="button"
            >
              {{ $t("buttons.cancel") }}
            </button>
          </router-link>
          <button class="button" type="submit">
            <i class="material-icons">save</i>
            {{ $t("buttons.save") }}
          </button>
        </div>
      </section>
    </form>
  </div>
</template>

<script setup lang="ts">
import { useAuthStore } from "@/stores/auth";
import { useLayoutStore } from "@/stores/layout";
import { users as api, settings } from "@/api";
import UserForm from "@/components/settings/UserForm.vue";
import Errors from "@/views/Errors.vue";
import { computed, inject, onMounted, ref, watch } from "vue";
import { useRoute, useRouter } from "vue-router";
import { useI18n } from "vue-i18n";
import { StatusError } from "@/api/utils";
import { authMethod } from "@/utils/constants";
import { logout } from "@/utils/auth";

const error = ref<StatusError>();
const originalUser = ref<IUser>();
const user = ref<IUser>();
const createUserDir = ref<boolean>(false);
const isCurrentPasswordRequired = ref<boolean>(false);

const $showError = inject<IToastError>("$showError")!;
const $showSuccess = inject<IToastSuccess>("$showSuccess")!;

const authStore = useAuthStore();
const layoutStore = useLayoutStore();
const route = useRoute();
const router = useRouter();
const { t } = useI18n();

onMounted(() => {
  fetchData();
});

const isNew = computed(() => route.path === "/settings/users/new");

watch(route, () => fetchData());
watch(user, () => {
  if (!user.value?.perm.admin) return;
  user.value.lockPassword = false;
});

const fetchData = async () => {
  layoutStore.loading = true;

  try {
    if (isNew.value) {
      const { defaults, createUserDir: _createUserDir } = await settings.get();
      isCurrentPasswordRequired.value = authMethod == "json";
      createUserDir.value = _createUserDir;
      user.value = {
        ...defaults,
        username: "",
        password: "",
        rules: [],
        lockPassword: false,
        id: 0,
      };
    } else {
      const { authMethod } = await settings.get();
      isCurrentPasswordRequired.value = authMethod == "json";
      const id = Array.isArray(route.params.id)
        ? route.params.id.join("")
        : route.params.id;
      user.value = { ...(await api.get(parseInt(id))) };
    }
  } catch (err) {
    if (err instanceof Error) {
      error.value = err;
    }
  } finally {
    layoutStore.loading = false;
  }
};

const deletePrompt = () => {
  if (isCurrentPasswordRequired.value) {
    layoutStore.showHover({
      prompt: "current-password",
      confirm: (event: Event, currentPassword: string) => {
        event.preventDefault();
        layoutStore.closeHovers();
        deleteUser(currentPassword);
      },
    });
  } else {
    layoutStore.showHover({
      prompt: "deleteUser",
      confirm: () => deleteUser(""),
    });
  }
};

const deleteUser = async (currentPassword: string) => {
  if (!user.value) {
    return false;
  }
  try {
    await api.remove(user.value.id, currentPassword);
    if (user.value.id == authStore.user?.id) {
      logout();
    } else {
      router.push({ path: "/settings/users" });
    }
    $showSuccess(t("settings.userDeleted"));
  } catch (err) {
    if (err instanceof StatusError) {
      err.status === 403 ? $showError(t("errors.forbidden")) : $showError(err);
    } else if (err instanceof Error) {
      $showError(err);
    }
  }

  return true;
};

const save = (event: Event) => {
  event.preventDefault();
  if (isCurrentPasswordRequired.value) {
    layoutStore.showHover({
      prompt: "current-password",
      confirm: (event: Event, currentPassword: string) => {
        event.preventDefault();
        layoutStore.closeHovers();
        send(currentPassword);
      },
    });
  } else {
    send("");
  }

  return true;
};

const send = async (currentPassword: string) => {
  if (!user.value) {
    return false;
  }

  try {
    if (isNew.value) {
      const newUser: IUser = {
        ...originalUser?.value,
        ...user.value,
      };

      const loc = await api.create(newUser, currentPassword);
      router.push({ path: loc || "/settings/users" });
      $showSuccess(t("settings.userCreated"));
    } else {
      await api.update(user.value, ["all"], currentPassword);

      if (user.value.id === authStore.user?.id) {
        authStore.updateUser(user.value);
      }

      $showSuccess(t("settings.userUpdated"));
    }
  } catch (e: any) {
    $showError(e);
  }
};
</script>
