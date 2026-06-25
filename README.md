# slidev-component-progress

[![NPM version](https://img.shields.io/npm/v/slidev-component-progress?color=3AB9D4&label=)](https://www.npmjs.com/package/slidev-component-progress)

Progress component for Slidev `Slidev`.
Shows a progress bar that represents the completed percentage of the presentation.

![Progress demo](./assets/progress.gif)

## Installation

```bash
npm i slidev-component-progress
```

## Configuration

Define this package into your slidev addons.

In your slides metadata (using frontmatter):
```
---
addons:
  - slidev-component-progress
---
```

Or in your `package.json`:
```json
{
  "slidev": {
    "addons": [
      "slidev-component-progress"
    ]
  }
}
```

## Usage

Create a `./global-top.vue` file in your `Slidev` project and use the component:
```vue
<template>
  <Progress level="2">
    <template #marker="args"><MarkersShapes v-bind="args" /></template>
  </Progress>
</template>
```

The `marker` slot renders the shape drawn at each stop on the bar. Use the
provided `MarkersShapes` component (see below), or supply your own.

## Components

### Progress

Component that displays the slides progress:
```vue
<Progress level="2"/>
```

Parameters:

* `activeColor` (type: `string`, default: `'#ffffff'` (light) or `'#000000'` (dark)): The color of the active item
* `strokeColor` (type: `string`, default: `'#000000'` (light) or `'#ffffff'` (dark)): The stroke color for the items
* `barColor` (type: `string`): The color of the progress bar, by default it uses the `--slidev-theme-primary` CSS variable
* `height` (type: `number`, default: `level * 4`): Height of the progress bar in pixels
* `level` (type: `number | string`, default: `1`): Maximum heading level shown in the progress bar, using information from the Table Of Content
* `marginX` (type: `string`, default: `'2px'`): Horizontal margin around the bar (any CSS length); used as the default for `marginL` and `marginR`
* `marginL` (type: `string`, default: `marginX`): Left margin around the bar (any CSS length)
* `marginR` (type: `string`, default: `marginX`): Right margin around the bar (any CSS length)
* `marginY` (type: `string`, default: `'2px'`): Vertical margin around the bar (any CSS length)
* `opacity` (type: `number | string`, default: `0.5`): Opacity of the progress bar when not hovered
* `position` (type: `'top' | 'bottom'`, default: `'top'`): Position of the progress bar in the slide
* `scale` (type: `'headings' | 'slides' | 'clicks'`, default: `'headings'`): How horizontal space is allocated and which slides get a marker.
  * `'headings'` (default): only slides with a heading get a marker. A heading-less slide is folded into the preceding heading — without `clicks` the bar simply stays put while you are on it; with `clicks` its clicks count towards the previous heading's segment and the bar advances through them.
  * `'slides'`: every slide gets an equal-width segment, including slides with no heading (their `marker` slot receives `level: 0`, and the bundled renderers draw nothing for them).
  * `'clicks'`: every slide's segment is sized proportionally to its number of clicks (a slide with more clicks takes up more of the bar). Slides with no clicks keep a base width; heading-less slides are treated like any other slide.
* `thickness` (type: `number | string`, default: `2px`): Thickness of the bar line. A number is interpreted as pixels; any CSS length string is used verbatim (e.g. `'0.2rem'`)
* `transitionDuration` (type: `string`, default: `200ms`): CSS transition duration
* `clicks` (type: `boolean`, default: `false`): How the bar advances. Markers are always one per slide (the largest heading of each slide).
  * unset (default): the bar advances one step per slide.
  * set: the bar advances per click (slide number + click number).
* `disable` (type: `(layout: string) => boolean`, default: hides the bar on the `cover` and `end` layouts): Predicate deciding whether the bar is hidden for a given slide layout
* `print` (type: `boolean`, default: `false`): When set, keep showing the bar while rendering for print/export (`?print` in the URL); by default the bar is hidden in print mode
* `fillLast` (type: `boolean`, default: `false`): Extend the bar to 100% on the very last click (last slide + last click)
* `emptyFirst` (type: `boolean`, default: `false`): Collapse the bar to 0% on the very first click (first slide + first click)
* `fade` (type: `boolean`, default: `false`): Fade the bar's leading edge with a gradient mask on the first slide and whenever the current slide's marker is hidden from the ToC

### MarkersShapes

Default renderer for the `marker` slot. Draws the shape for each stop on the bar
and highlights the active one. Bind the slot args with `v-bind="args"` and add
any of the options below:
```vue
<template #marker="args"><MarkersShapes v-bind="args" nested activeStyle="filled"/></template>
```

Parameters:

* `nested` (type: `boolean`, default: `false`): When set, draw one shape per heading level (nested shapes); otherwise draw a single shape
* `visible` (type: `'always' | 'hover' | 'never'`, default: `'always'`): When the markers are visible
  * `'always'`: always shown
  * `'hover'`: only shown while the `Progress` bar is hovered
  * `'never'`: never shown (markers still exist and stay clickable)
* `shape` (type: `'circle' | 'square'`, default: `'circle'`): Shape drawn for each marker
* `activeStyle` (type: `'outlined' | 'filled' | 'none'`, default: `'outlined'`): How the active marker is highlighted
  * `'outlined'`: a larger shape is drawn behind it
  * `'filled'`: the marker itself is filled with its stroke color
  * `'none'`: no special highlight on the active marker

The remaining props (`level`, `maxLevel`, `height`, `active`, `prev`, `next`)
are supplied by the `Progress` bar through the slot and are forwarded by
`v-bind="args"` — you don't set them yourself. `level` is the heading level
(`1` = top level); with `scale="slides"`/`"clicks"` a slide that has **no**
heading is passed `level: 0`, which you can use in a custom `marker` slot to
render heading-less slides differently (the bundled renderers draw nothing).
