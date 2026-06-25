<script setup lang="ts">
withDefaults(
  defineProps<{
    /* properties specified by the user */
    /** true => multiple shapes / false => a single one */
    nested?: boolean;
    /**
     * when the markers should be visible:
     * - "always": always shown
     * - "hover": only shown while the `.progress` bar is hovered
     * - "never": never shown (markers still exist and stay clickable)
     */
    visible?: "always" | "hover" | "never";
    /** shape to draw for each marker */
    shape?: "circle" | "square";
    /**
     * how the active marker is highlighted:
     * - "outlined": a larger shape is drawn behind it (default)
     * - "filled": the marker itself is filled with its stroke color
     * - "none": no special highlight on active
     */
    activeStyle?: "outlined" | "filled" | "none";

    /* properties passed from the bar itself */
    /** level of this marker (1 = top level, 0 = slide without a heading). */
    level: number;
    /** deepest level across all markers. */
    maxLevel: number;
    /** height specified for the progress bar itself. */
    height: number;
    /** marker is the current route. */
    active?: boolean;
    /** marker has already been passed. */
    prev?: boolean;
    /** marker is still upcoming. */
    next?: boolean;
  }>(),
  { // defaults
    nested: false,
    visible: "always",
    shape: "circle",
    activeStyle: "outlined",
  }
);
</script>

<template>
  <svg
    class="markers-shapes block overflow-visible transition-opacity duration-[var(--transition-duration)]"
    :class="`markers-shapes--${visible}`"
    :width="height"
    :height="height"
  >
    <!-- heading-less slide (level 0): nothing is drawn. -->
    <template v-if="level < 1" />
    <template v-else-if="shape === 'circle'">
      <!-- active outline: a larger shape drawn behind to highlight the current marker -->
      <circle
        v-if="active && activeStyle === 'outlined'"
        fill="var(--bar-color)"
        :r="Math.max(0, height / 2 - (level - 1) * 2 - 1) + 2"
        :cx="height / 2"
        :cy="height / 2"
      />
      <circle
        v-for="circle in nested ? maxLevel - level + 1 : 1"
        :key="circle"
        fill="transparent"
        class="stroke-[var(--stroke-color)] transition-[fill] duration-[var(--transition-duration)]"
        :class="{
          'fill-[var(--active-color)]': next,
          'fill-[var(--bar-color)]': prev || (active && activeStyle === 'outlined'),
          'fill-[var(--stroke-color)]': active && activeStyle === 'filled',
        }"
        :r="Math.max(0, (height / 2 - (level + circle - 2) * 2) - 1)"
        :cx="height / 2"
        :cy="height / 2"
      /><!-- r needs "- 1" to account for the stroke width -->
    </template>
    <template v-else>
      <!-- active outline: a larger shape drawn behind to highlight the current marker -->
      <rect
        v-if="active && activeStyle === 'outlined'"
        fill="var(--bar-color)"
        :x="(level - 1) * 2 + 1 - 2"
        :y="(level - 1) * 2 + 1 - 2"
        :width="Math.max(0, height - (level - 1) * 4 - 2) + 4"
        :height="Math.max(0, height - (level - 1) * 4 - 2) + 4"
      />
      <rect
        v-for="rect in nested ? maxLevel - level + 1 : 1"
        :key="rect"
        fill="transparent"
        class="stroke-[var(--stroke-color)] transition-[fill] duration-[var(--transition-duration)]"
        :class="{
          'fill-[var(--active-color)]': next,
          'fill-[var(--bar-color)]': prev || (active && activeStyle === 'outlined'),
          'fill-[var(--stroke-color)]': active && activeStyle === 'filled',
        }"
        :x="(level + rect - 2) * 2 + 1"
        :y="(level + rect - 2) * 2 + 1"
        :width="Math.max(0, height - (level + rect - 2) * 4 - 2)"
        :height="Math.max(0, height - (level + rect - 2) * 4 - 2)"
      /><!-- inset by 1 on each side to account for the stroke width -->
    </template>
  </svg>
</template>

<style>
/* "never": hidden but still rendered, so the surrounding link stays clickable. */
.markers-shapes--never {
  opacity: 0;
}
/* "hover": only revealed while the progress bar is hovered. */
.markers-shapes--hover {
  opacity: 0;
  will-change: opacity;
}
.progress:hover .markers-shapes--hover {
  opacity: 1;
}
</style>
