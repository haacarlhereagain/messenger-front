<template>
  <q-layout
    view="lHh Lpr lFf"
    class="flex column main-wrapper"
  >
    <router-view v-if="!!isAppReady" />
  </q-layout>
</template>

<script lang="ts">
import {computed, defineComponent, onBeforeMount} from 'vue';
import { useStore } from 'vuex';
import { getUserData } from '@/api/user.api';
import { useRoute, useRouter } from 'vue-router';
import { useQuasar } from 'quasar';
import '@/api/messenger.api';

export default defineComponent({
  name: 'App',
  setup() {
    const $store = useStore();
    const $route = useRoute();
    const $router = useRouter();
    const $quasar = useQuasar();

    onBeforeMount(async () => {
      try {
        $quasar.loading.show();
        await $router.isReady();
        const user = await getUserData();
        user && $store.commit('userModule/setUserData', user);
        await $router.push({ name: 'Main' });
      } catch (e) {
        $route.name !== 'Register' && await $router.push({ name: 'Login' });
      } finally {
        $store.commit('setIsDataLoaded');
        $quasar.loading.hide();
      }
    });

    const isAppReady = computed(() => $store.getters['isDataLoaded']);

    return {
      isAppReady
    }
  }
});
</script>

<style lang="scss">
.main-wrapper {
  max-height: 100vh;
  overflow: hidden;
  flex-wrap: nowrap!important;
}
</style>
