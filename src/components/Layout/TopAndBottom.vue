<script setup lang="ts">
import { computed } from 'vue';

const browserHasHistory = computed(() => window.history.state.back);
</script>

<template>
  <div class="section mt-4">
    <div class="container-default">
      <div>
        <div class="integration-content">
          <slot />
          <div
            class="top-content card integration-main-card integration-card-top-content"
            v-if="$slots['top-content']"
          >
            <slot name="top-content" />
          </div>
          <div
            class="split-content integration-card-content"
            v-if="$slots['content-bottom']"
          >
            <slot name="content-bottom" />
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
