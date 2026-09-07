<template>
  <header>
    <div class="header-left">
      <!-- In Settings view: Back button + Settings title -->
      <template v-if="isSettings">
        <router-link
          to="/files/"
          class="header-back-btn"
          :title="t('buttons.back')"
          aria-label="Back to files"
        >
          <i class="material-icons">arrow_back</i>
        </router-link>
        <span class="settings-header-title">{{ t("sidebar.settings") }}</span>
      </template>

      <!-- Standard view: Hamburger menu + Brand Logo/Name -->
      <template v-else>
        <Action
          v-if="showMenu"
          class="menu-button"
          icon="menu"
          :label="t('buttons.toggleSidebar')"
          @action="layoutStore.showHover('sidebar')"
        />
        <router-link to="/files/" class="brand-link" v-if="showLogo">
          <img :src="logoURL" :alt="brandName" class="brand-logo" />
          <span class="brand-name">{{ brandName }}</span>
        </router-link>
      </template>
    </div>

    <div class="header-center">
      <slot />
    </div>

    <div class="header-right">
      <!-- Settings Quick Link (Only when not already in settings) -->
      <router-link
        v-if="!isSettings"
        to="/settings"
        class="header-icon-btn"
        :title="t('sidebar.settings')"
        aria-label="Settings"
      >
        <i class="material-icons">settings</i>
      </router-link>

      <!-- User Profile / Account Link -->
      <router-link
        v-if="authStore.isLoggedIn"
        to="/settings/profile"
        class="header-icon-btn user-btn"
        :title="authStore.user?.username || t('settings.user')"
        aria-label="Profile"
      >
        <div class="user-avatar">
          {{ userInitial }}
        </div>
      </router-link>
    </div>

    <div
      class="overlay"
      v-show="layoutStore.currentPromptName == 'more'"
      @click="layoutStore.closeHovers"
    />
  </header>
</template>

<script setup lang="ts">
import { useLayoutStore } from "@/stores/layout";
import { useAuthStore } from "@/stores/auth";
import { logoURL, name as appName } from "@/utils/constants";
import Action from "@/components/header/Action.vue";
import { computed } from "vue";
import { useI18n } from "vue-i18n";
import { useRoute } from "vue-router";

defineProps<{
  showLogo?: boolean;
  showMenu?: boolean;
}>();

const layoutStore = useLayoutStore();
const authStore = useAuthStore();
const route = useRoute();
const { t } = useI18n();

const isSettings = computed(() => route.path.startsWith("/settings"));
const brandName = computed(() => appName || "File Browser");
const userInitial = computed(() => {
  if (authStore.user?.username) {
    return authStore.user.username.charAt(0).toUpperCase();
  }
  return "U";
});
</script>
