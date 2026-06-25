<script setup lang="ts">
import { computed } from "vue";

// Drop-in alternative to `MarkersShapes.vue` with an identical prop contract,
// but rendered with CSS-styled `<span>`s instead of SVG. Firefox paints plain
// CSS boxes / border-radius noticeably cheaper than SVG shapes, so swapping this
// in keeps click-through smooth on large decks. See `MarkersShapes.vue` for the
// authoritative geometry this mirrors.
const props = withDefaults(
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

// Fill color for the marker shape, mirroring MarkersShapes.vue's class logic:
// next => active-color, prev / active-outlined => bar-color,
// active-filled => stroke-color, otherwise transparent (stroke ring only).
const fillColor = computed(() => {
  if (props.next) return "var(--active-color)";
  if (props.prev || (props.active && props.activeStyle === "outlined")) return "var(--bar-color)";
  if (props.active && props.activeStyle === "filled") return "var(--stroke-color)";
  return "transparent";
});

type Shape = { size: number; offset: number };

// Concentric shapes, outermost first so the smaller ones paint on top and their
// borders read as inner rings — same draw order as the SVG version. `size` is
// the box edge length; `offset` is the top/left inset within the height×height
// container. Geometry mirrors the SVG radii/coords, plus 1px on the edge length
// (offset shifted 0.5px to stay centered): the SVG's centered 1px stroke paints
// half outside the radius, so it reads 1px wider than a border-box box would.
const shapes = computed<Shape[]>(() => {
  const h = props.height;
  // Heading-less slide (level 0): nothing is drawn.
  if (props.level < 1) return [];
  const count = props.nested ? props.maxLevel - props.level + 1 : 1;
  const out: Shape[] = [];
  for (let k = 1; k <= count; k++) {
    if (props.shape === "circle") {
      const r = Math.max(0, h / 2 - (props.level + k - 2) * 2 - 1);
      out.push({ size: 2 * r + 1, offset: h / 2 - r - 0.5 });
    } else {
      out.push({
        size: Math.max(0, h - (props.level + k - 2) * 4 - 2) + 1,
        offset: (props.level + k - 2) * 2 + 1 - 0.5,
      });
    }
  }
  return out;
});

// Larger shape drawn behind to highlight the active marker (outlined style only).
const outline = computed<Shape | null>(() => {
  if (!(props.active && props.activeStyle === "outlined")) return null;
  const h = props.height;
  // Heading-less slide (level 0): nothing is drawn.
  if (props.level < 1) return null;
  if (props.shape === "circle") {
    const r = Math.max(0, h / 2 - (props.level - 1) * 2 - 1) + 2;
    return { size: 2 * r, offset: h / 2 - r };
  }
  return {
    size: Math.max(0, h - (props.level - 1) * 4 - 2) + 4,
    offset: (props.level - 1) * 2 + 1 - 2,
  };
});

const radius = computed(() => (props.shape === "circle" ? "50%" : "0"));

const shapeStyle = (s: Shape) => ({
  width: `${s.size}px`,
  height: `${s.size}px`,
  top: `${s.offset}px`,
  left: `${s.offset}px`,
  borderRadius: radius.value,
});
</script>

<template>
  <div
    class="markers-dots"
    :class="`markers-dots--${visible}`"
    :style="{ width: `${height}px`, height: `${height}px` }"
  >
    <span
      v-if="outline"
      class="markers-dots__outline"
      :style="shapeStyle(outline)"
    />
    <span
      v-for="(s, k) in shapes"
      :key="k"
      class="markers-dots__shape"
      :style="{ ...shapeStyle(s), backgroundColor: fillColor }"
    />
  </div>
</template>

<style>
.markers-dots {
  position: relative;
  display: block;
  overflow: visible;
  transition: opacity var(--transition-duration);
}

.markers-dots__shape,
.markers-dots__outline {
  position: absolute;
  box-sizing: border-box;
}

.markers-dots__shape {
  border: 1px solid var(--stroke-color);
  transition: background-color var(--transition-duration);
}

.markers-dots__outline {
  background-color: var(--bar-color);
}

/* "never": hidden but still rendered, so the surrounding link stays clickable. */
.markers-dots--never {
  opacity: 0;
}
/* "hover": only revealed while the progress bar is hovered. */
.markers-dots--hover {
  opacity: 0;
  will-change: opacity;
}
.progress:hover .markers-dots--hover {
  opacity: 1;
}
</style>
