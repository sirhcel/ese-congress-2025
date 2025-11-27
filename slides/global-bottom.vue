<template>
  <footer v-if="$slidev.nav.currentLayout !== 'cover'" class="absolute bottom-0 right-0 p-2 text-gray-400">
    <SlideCurrentNo/>/<SlidesTotal/>
  </footer>
</template>
