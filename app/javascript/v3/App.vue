<script>
import SnackbarContainer from './components/SnackBar/Container.vue';

export default {
  components: { SnackbarContainer },
  data() {
    return { theme: 'light' };
  },
  mounted() {
    this.setColorTheme();
    this.listenToThemeChanges();
    this.setLocale(window.chatwootConfig.selectedLocale);
  },
  methods: {
    setColorTheme() {
      if (window.matchMedia('(prefers-color-scheme: dark)').matches) {
        this.theme = 'dark';
      } else {
        this.theme = 'light ';
      }
    },
    listenToThemeChanges() {
      const mql = window.matchMedia('(prefers-color-scheme: dark)');

      mql.onchange = e => {
        if (e.matches) {
          this.theme = 'dark';
        } else {
          this.theme = 'light';
        }
      };
    },
    setLocale(locale) {
      this.$root.$i18n.locale = locale;
    },
  },
};
</script>

<template>
  <div
    class="h-full min-h-screen w-full antialiased bg-n-background text-n-slate-12 relative isolate before:content-[''] before:absolute before:-inset-x-[10%] before:-top-[10%] before:bottom-[40%] before:[background:radial-gradient(800px_360px_at_20%_0%,rgba(34,211,238,.22),transparent_55%),_radial-gradient(900px_420px_at_80%_0%,rgba(37,99,235,.18),transparent_60%)] before:pointer-events-none before:-z-10"
    :class="theme"
  >
    <router-view />
    <SnackbarContainer />
  </div>
</template>

<style lang="scss">
@tailwind base;
@tailwind components;
@tailwind utilities;

@import '../dashboard/assets/scss/next-colors';

html,
body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto,
    Oxygen-Sans, Ubuntu, Cantarell, 'Helvetica Neue', sans-serif;
  @apply h-full w-full;

  input,
  select {
    outline: none;
  }
}

.text-link {
  @apply text-n-brand font-medium hover:text-n-blue-10;
}

.v-popper--theme-tooltip .v-popper__inner {
  background: black !important;
  font-size: 0.75rem;
  padding: 4px 8px !important;
  border-radius: 6px;
  font-weight: 400;
}

.v-popper--theme-tooltip .v-popper__arrow-container {
  display: none;
}
</style>
