<script setup lang="ts">
import { computed, ref } from "vue";
import { useNav, useSlideContext } from "@slidev/client"

import type { TocItem } from "@slidev/types";

import Titles from "#slidev/title-renderer";

const { $nav } = useSlideContext()
const { isPrintMode } = useNav()
// Use `$nav.value` rather than `useNav()`
// to make it work in print/export mode. See https://github.com/slidevjs/slidev/issues/2310.
const currentPage = computed(() => $nav.value.currentPage)
const currentLayout = computed(() => $nav.value.currentLayout)
const tocTree = computed(() => $nav.value.tocTree)
const total = computed(() => $nav.value.total)
const currentClicks = computed(() => $nav.value.clicks)
const clicksTotal = computed(() => $nav.value.clicksTotal)
const slides = computed(() => $nav.value.slides)

const props = defineProps<{
  activeColor?: string;
  strokeColor?: string;
  barColor?: string;
  /** height of the progress bar (in pixels) */
  height?: number;
  /** maximum level of headings displayed */
  level?: number | string;
  /** horizontal margin around the bar (CSS length); default for marginL/marginR */
  marginX?: string;
  /** left margin around the bar (CSS length); defaults to marginX */
  marginL?: string;
  /** right margin around the bar (CSS length); defaults to marginX */
  marginR?: string;
  /** vertical margin around the bar (CSS length) */
  marginY?: string;
  opacity?: number | string;
  position?: "top" | "bottom";
  /**
   * How the bar allocates horizontal space and which slides get a marker:
   * - `"headings"` (default): only slides with a heading get a marker. A
   *   heading-less slide is folded into the preceding heading: without `clicks`
   *   the bar simply stays put while you are on it; with `clicks` its clicks
   *   count towards the previous heading's segment and the bar advances through
   *   them.
   * - `"slides"`: every slide gets an equal-width segment, including slides that
   *   have no heading (their marker slot receives `level: 0` and, by default,
   *   renders nothing).
   * - `"clicks"`: every slide gets a segment sized proportionally to its number
   *   of clicks (a slide with more clicks takes up more of the bar). Heading-less
   *   slides are treated like any other slide.
   */
  scale?: "slides" | "clicks" | "headings";
  /**
   * Thickness of the bar line.
   * a number (pixels) or a string used verbatim as CSS length (e.g. `"0.2rem"`)
   */
  thickness?: number | string;
  transitionDuration?: string;
  /** bar advances per click */
  clicks?: boolean;
  /**
   * Predicate deciding whether the bar is hidden for a given layout.
   * Defaults to hiding the bar on the `cover` and `end` layouts.
   */
  disable?: (layout: string) => boolean;
  /** hide the bar when rendering for print/export (`?print` in the URL) */
  print?: boolean;
  /** extend the bar to 100% on the very last click (last slide + last click) */
  fillLast?: boolean;
  /** collapse the bar to 0% on the very first click (first slide + first click) */
  emptyFirst?: boolean;
  /**
   * Fade the bar's leading edge with a gradient mask on the first slide and
   * whenever the current slide's marker is hidden from the ToC. Defaults to `true`.
   */
  fade?: boolean;
}>();

const {
  opacity = 0.5,

  fillLast = false,
  emptyFirst = false,
  print = false,
  disable = (layout: string) => new Set(["cover", "end"]).has(layout),
  level = 1,
  clicks = false,
  fade = false,
  scale = "headings",
} = props;

// Flatten the ToC tree depth-first into a single list of nodes. Slidev builds
// `tocTree` with one node per slide that has a heading (nesting them by heading
// level), so this yields the heading-bearing slides in slide order.
const flatten = (tree: TocItem[]): TocItem[] => tree.reduce<TocItem[]>(
  (acc, item) => acc.concat(item, flatten(item.children ?? [])),[]
);

// Markers for slides that carry a heading (the ToC nodes), in slide order.
const headingMarkers = computed(() => flatten(tocTree.value));

// One marker per slide, in slide order. Slides that have no ToC heading get a
// synthetic marker with `level: 0` — the sentinel the `marker` slot receives so
// it can render heading-less slides differently. The tooltip still resolves the
// real title (if any) through `Titles`.
const allMarkers = computed<TocItem[]>(() => {
  const byNo = new Map(headingMarkers.value.map((m) => [m.no, m] as const));
  return slides.value.map((route) => {
    const existing = byNo.get(route.no);
    if (existing) return existing;
    const slide = route.meta?.slide as
      | { level?: number; title?: string; frontmatter?: { routeAlias?: string; hideInToc?: boolean } }
      | undefined;
    return { // some fields only exist to satisfy `TocItem`
      no: route.no,
      path: `/${route.no}`,
      level: 0,
      titleLevel: slide?.level ?? 0,
      children: [],
      title: slide?.title,
      hideInToc: Boolean(slide?.frontmatter?.hideInToc),
    } satisfies TocItem;
  });
});

// `"headings"` scale keeps only heading-bearing slides as markers; `"slides"`
// and `"clicks"` give every slide its own marker.
const markers = computed(() =>
  scale === "headings" ? headingMarkers.value : allMarkers.value
);

// Total clicks Slidev knows about for a given slide number. `__clicksContext`
// is populated once a slide has been rendered; before that we fall back to a
// `clicks` count declared in the slide's frontmatter, and finally to 0.
function clicksOf(no: number): number {
  const meta = slides.value[no - 1]?.meta as
    | { __clicksContext?: { total?: number } | null; clicks?: number }
    | undefined;
  return meta?.__clicksContext?.total ?? meta?.clicks ?? 0;
}

// Range of slide numbers [start, end] covered by marker `i`'s segment. For
// `"slides"`/`"clicks"` this is a single slide; for `"headings"` it also absorbs
// the run of heading-less slides up to the next heading.
function groupRange(i: number): [number, number] {
  const start = markers.value[i].no;
  const next = markers.value[i + 1];
  return [start, (next ? next.no : total.value + 1) - 1];
}

// Relative width of each marker's segment. `"clicks"` weights a segment by its
// slide's clicks + 1 (so the slide-entry step keeps a base width and each click
// adds a unit); `"slides"` and `"headings"` give every marker an equal segment.
const weights = computed(() =>
  markers.value.map((item) => (scale === "clicks" ? clicksOf(item.no) + 1 : 1))
);
const totalWeight = computed(() => weights.value.reduce((sum, w) => sum + w, 0));

// Cumulative weight preceding each marker — i.e. the fractional position of
// each marker's segment start, before dividing by `totalWeight`.
const cumBefore = computed(() => {
  const out: number[] = [];
  let sum = 0;
  for (const w of weights.value) {
    out.push(sum);
    sum += w;
  }
  return out;
});

// Progress within the current marker's segment as a fraction (0..1). Only
// click-aware when `clicks` is set; otherwise the bar rests at the segment
// start. Every slide in the segment contributes `clicks + 1` steps, so the
// clicks of any folded-in heading-less slides advance the bar too.
const withinFraction = computed(() => {
  if (!clicks) return 0;
  const [start, end] = groupRange(lastActiveIndex.value);
  const cur = currentPage.value;
  let reached = 0;
  let groupWeight = 0;
  for (let s = start; s <= end; s++) {
    groupWeight += clicksOf(s) + 1;
    if (s < cur) reached += clicksOf(s) + 1;
    else if (s === cur) reached += Math.min(currentClicks.value, clicksOf(s));
  }
  return groupWeight > 0 ? Math.min(reached / groupWeight, 1) : 0;
});

// Bar end as a fraction (0..1) of the container width: the start of the current
// segment plus the progress made within it.
const barFraction = computed(() => {
  const n = markers.value.length;
  if (n === 0 || totalWeight.value === 0) return 0;
  const i = lastActiveIndex.value;
  return (cumBefore.value[i] + withinFraction.value * weights.value[i]) / totalWeight.value;
});

// Continuous count of segments filled, used only to size the fade gradient.
const barSegments = computed(() => lastActiveIndex.value + withinFraction.value);

// True on the very last click of the presentation (last slide, last click).
const isVeryLast = computed(() =>
  currentPage.value === total.value &&
  currentClicks.value >= clicksTotal.value
);

// True on the very first click of the presentation (first slide, first click).
const isVeryFirst = computed(() =>
  currentPage.value === 1 &&
  currentClicks.value <= 0
);

const disabled = computed(
  () => markers.value.length === 0
    || disable(currentLayout.value)
    || (!print && isPrintMode.value)
);

const height = computed(() =>
  props.height != null ? props.height : Number(level) * 4
);

const thickness = computed(() =>
  typeof props.thickness === "number" ? props.thickness + "px" : props.thickness
);

const cssVars = computed(() =>
  // Drop unset values so component defaults / the stylesheet win.
  Object.fromEntries(Object.entries({
    "--active-color": props.activeColor,
    "--bar-color": props.barColor,
    "--height": height.value + "px",
    "--margin-l": props.marginL || props.marginX || "2px",
    "--margin-r": props.marginR || props.marginX || "2px",
    "--margin-y": props.marginY || "2px",
    "--opacity": opacity,
    "--stroke-color": props.strokeColor,
    "--thickness": thickness.value,
    "--transition-duration": props.transitionDuration,
    display: disabled.value ? "none" : undefined,
  }).filter(([, value]) => value))
);

// Index of the furthest marker at or before the current slide. Because markers
// are in slide order, this is the current slide's own marker in `"slides"`/
// `"clicks"` scale, and the most recent heading in `"headings"` scale (so a
// heading-less slide keeps the bar on the preceding heading's segment).
const lastActiveIndex = computed(() => {
  const no = currentPage.value ?? 0;
  let last = 0;
  markers.value.forEach((item, i) => {
    if (item.no <= no) last = i;
  });
  return last;
});

const isVisible = (item: TocItem) => item.level <= Number(level);

const activeIndex = computed(() => {
  for (let i = lastActiveIndex.value; i >= 0; i--) {
    if (isVisible(markers.value[i])) return i;
  }
  return lastActiveIndex.value;
});

function goTo(event: MouseEvent, no: number): void {
  event.preventDefault();
  // Drop focus
  (event.currentTarget as HTMLElement | null)?.blur();
  $nav.value.go(no);
}

// Lazily mount tooltip titles.
const activated = ref(new Set<number>());

const gradientWidth = computed(() =>
  barSegments.value <= 0
    ? "0%"
    : `(100% - var(--height) / 2 - var(--margin-l)) / ${barSegments.value}`
);
</script>

<style>
.dark .progress {
  --bar-color: var(--slidev-theme-primary, #ffffff);
  --stroke-color: #ffffff;
  --active-color: #000000;
}
.progress {
  --bar-color: var(--slidev-theme-primary, #000000);
  --stroke-color: #000000;
  --active-color: #ffffff;

  --thickness: 2px;
  --transition-duration: 200ms;
  --line-height: 1.5em;
  --padding: 8px; /* padding between markers and tooltips */
  --tip-shift: 0.5; /* per-marker shift (0..1); centered by default */
  /* edge correction so the first/last glyph (not its outer edge) centers under the bar */
  --tip-edge: 0.3em;

  position: absolute;
  top: 0;
  right: 0;
  left: 0;
  overflow: visible;
  font-size: 80%;
  line-height: var(--line-height);
  opacity: var(--opacity);
  transition:
    opacity var(--transition-duration),
    background-color var(--transition-duration);

  &:hover {
    background-color: inherit;
    opacity: 1;
  }
}

.progress--bottom {
  top: initial;
  bottom: 0;
}

.progress__container {
  height: var(--height);
  margin: var(--margin-y) var(--margin-r) var(--margin-y) var(--margin-l);
  display: grid;
}

.progress__bar {
  grid-area: 1 / 1;
  align-self: center;
  justify-self: start;
  height: var(--thickness);
  margin: 0 calc(var(--margin-r) * -1) 0 calc(var(--margin-l) * -1);
  width: calc(
    var(--height) / 2 + var(--bar-fraction, 0) * 100% + var(--margin-l)
  );
  background-color: var(--bar-color);
  transition: width var(--transition-duration);
}

/* With `fillLast`, the final click fills the bar to the container's right edge. */
.progress__bar--fill {
  width: calc(100% + var(--margin-l) + var(--margin-r));
}

/* With `emptyFirst`, the first click collapses the bar to nothing. */
.progress__bar--empty {
  width: 0;
}

.progress__bar--next {
  -webkit-mask-image: linear-gradient(
    to left,
    transparent,
    rgba(0, 0, 0, 1) calc(var(--gradient-width)),
    rgba(0, 0, 0, 1) 100%
  );
  mask-image: linear-gradient(
    to left,
    transparent,
    rgba(0, 0, 0, 1) calc(var(--gradient-width)),
    rgba(0, 0, 0, 1) 100%
  );
}

.progress__markers {
  grid-area: 1 / 1;
  z-index: 1;
  display: flex;
  align-items: stretch;
}

.progress__link {
  /* `--flex-grow` weights each slot: 1 for the default per-slide scale, or the
   * slide's click count (+1) for the `"clicks"` scale. */
  flex: var(--flex-grow, 1) 1 0;
  min-width: 0;
  display: block;
  position: relative;
  contain: layout;
  &::before {
    content: "";
    position: absolute;
    top: calc(100% + var(--padding) * 0.1);
    left: calc(var(--height) / 2);
    width: 2px;
    /* 90% of the gap, leaving 10% space between the bar. */
    height: calc(var(--padding) * 0.9);
    border-radius: 2px;
    background-color: var(--stroke-color);
    box-shadow: 0 0 0 1px color-mix(in srgb, var(--active-color) 75%, transparent);
    /* no vertical translate s.t. it does not interfere with scale */
    transform: translateX(-50%);
    transform-origin: top center;
    scale: 1 0;
    visibility: hidden;
    opacity: 0;
    transition:
      visibility var(--transition-duration),
      opacity var(--transition-duration),
      scale var(--transition-duration) cubic-bezier(0.4, 0, 0.2, 1);
  }
}

/* The glyph is taken out of flow so its width can't push the slot wider; a
 * crowded bar lets glyphs overlap instead of overflowing the row. */
.progress__marker {
  position: absolute;
  top: 0;
  left: 0;
}

/* Stable hover bridge: `padding-top` keeps `:hover` continuous from bar to
 * tooltip. No animated transform/scale here, so the hit area never shifts
 * mid-animation; the entrance animation lives on `.progress__tooltip-content`
 * instead (animating it here would break the bridge and flicker). */
.progress__tooltip {
  position: absolute;
  width: max-content;
  left: calc(var(--height) / 2);
  bottom: 0;
  padding-top: var(--padding);
  translate: calc(var(--tip-shift) * -100% + (2 * var(--tip-shift) - 1) * var(--tip-edge))
    100%;
  white-space: nowrap;
  visibility: hidden;
  transition: visibility var(--transition-duration);
}

.progress__tooltip-content {
  display: inline-block;
  color: var(--stroke-color);
  text-shadow:
    -1px -1px 1px var(--active-color),  1px -1px 1px var(--active-color),
    -1px  1px 1px var(--active-color),  1px  1px 1px var(--active-color),
     0   -1px 1px var(--active-color),  0    1px 1px var(--active-color),
    -1px  0   1px var(--active-color),  1px  0   1px var(--active-color);
  opacity: 0;
  scale: 0.94;
  transform: translateY(-4px);
  transform-origin: calc(var(--tip-shift) * 100%) top;
  transition:
    opacity var(--transition-duration),
    transform var(--transition-duration) cubic-bezier(0.2, 0.8, 0.2, 1.3),
    scale var(--transition-duration) cubic-bezier(0.2, 0.8, 0.2, 1.3);
}

/* The connector stem is the tooltip's stem: only reveal it when the marker
 * actually has a tooltip (a title). */
.progress__link--titled:hover::before,
.progress__link--visible:hover .progress__tooltip {
  visibility: visible;
}

.progress__link--titled:hover::before {
  opacity: 1;
  scale: 1 1;
}
.progress__link--visible:hover .progress__tooltip-content {
  opacity: 1;
  transform: translateY(0);
  scale: 1;
}

.progress--bottom .progress__link::before {
  top: auto;
  bottom: calc(100% + var(--padding) * 0.1);
  transform: translateX(-50%);
  transform-origin: bottom center;
}

.progress--bottom .progress__tooltip {
  top: 0;
  bottom: auto;
  padding-top: 0;
  padding-bottom: var(--padding);
  translate: calc(var(--tip-shift) * -100% + (2 * var(--tip-shift) - 1) * var(--tip-edge))
    -100%;
}

.progress--bottom .progress__tooltip-content {
  transform: translateY(4px);
  transform-origin: calc(var(--tip-shift) * 100%) bottom;
}

.progress--bottom .progress__link--visible:hover .progress__tooltip-content {
  transform: translateY(0);
}
</style>

<template>
  <div
    class="progress"
    :class="{ 'progress--bottom': position === 'bottom' }"
    :style="cssVars"
  >
    <div class="progress__container">
      <div
        class="progress__bar"
        :class="{
          'progress__bar--next': fade && activeIndex !== lastActiveIndex,
          'progress__bar--fill': fillLast && isVeryLast,
          'progress__bar--empty': emptyFirst && isVeryFirst,
        }"
        :style="{ '--bar-fraction': barFraction, '--gradient-width': gradientWidth }"
      ></div>
      <div class="progress__markers">
        <a
          v-for="(item, i) in markers" :key="item.path"
          v-memo="[i === activeIndex, i < activeIndex, i > activeIndex, activated.has(i), weights[i]]"
          class="progress__link"
          :class="{
            'progress__link--visible': isVisible(item),
            'progress__link--titled': isVisible(item) && !!item.title,
          }"
          :style="{
            '--tip-shift': markers.length > 1 ? i / (markers.length - 1) : 0.5,
            '--flex-grow': weights[i],
          }"
          :href="item.path"
          @click="goTo($event, item.no)"
          @mouseenter="activated.add(i)"
        >
          <template v-if="isVisible(item)">
            <div class="progress__marker">
              <slot
                name="marker"
                :item="item"
                :level="item.level"
                :max-level="Number(level)"
                :height="height"
                :active="i === activeIndex"
                :prev="i < activeIndex"
                :next="i > activeIndex"
              />
            </div>
            <div v-if="activated.has(i) && item.title" class="progress__tooltip">
              <span class="progress__tooltip-content">
                <Titles :no="item.no" />
              </span>
            </div>
          </template>
        </a>
      </div>
    </div>
  </div>
</template>
