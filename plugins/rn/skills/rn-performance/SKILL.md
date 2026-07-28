---
name: rn-performance
description: Reference for diagnosing and fixing React Native performance problems. Use when the app has slow lists, janky animations, slow startup, excessive re-renders, or when the user asks why the app feels sluggish. Covers FlashList, Reanimated, image handling, startup cost and profiling.
user-invocable: false
---

# Performance triage

## Two rules before touching code

1. Profile release builds. Dev mode runs a different engine configuration with dev-only checks; its performance characteristics are fiction. `npx expo run:ios --configuration Release` (or the Android equivalent) before believing any measurement.
2. Name the thread. Everything slow is one of: JS thread saturated, UI thread saturated, or waiting on IO. The fix lives on the thread that is busy, and the profiler tells you which one it is; guessing does not.

## Lists

- Long scrollable content belongs in FlashList (or at minimum FlatList), never a ScrollView with `.map()`. ScrollView mounts everything immediately; past a few dozen items that is the entire problem.
- renderItem must be a stable component reference, not an inline closure creating new component trees per render. Extract it, memo it, and keep item props primitive where possible.
- keyExtractor must return stable ids. Index keys plus insertions equals recycled rows showing the wrong content and pointless re-renders.
- Images inside rows: fixed dimensions, a recycling-aware image component, and correctly sized source assets. Decoding a 4000px image into a 80px avatar slot burns CPU and memory for nothing.
- Measure list health with the blank-area metric while scrolling fast, and by watching JS FPS in the perf monitor, not by feel on a simulator. Simulators lie in both directions.

## Re-renders

- Subscribe narrowly. With zustand and friends, select the smallest slice (`useStore(s => s.count)`), never the whole store object. With context, split fast-changing values out of the provider that wraps the world.
- memo the expensive leaves (list rows, chart bodies, anything with heavy trees), then make sure their props are actually stable, because memo with a fresh object or closure prop every render is a no-op with extra steps.
- useCallback and useMemo are for identity stability where a memoized child or an effect depends on it, not decoration for every function in the file.
- To find offenders, use the React DevTools profiler and look for wide flame bars re-rendering on interactions that should be local. One misplaced subscription at the screen root regularly accounts for the whole problem.

## Animations and gestures

- Reanimated worklets run on the UI thread; that is the entire point. An animation that calls back into JS every frame (runOnJS in a hot path, setState per frame) forfeits it.
- Animate transform and opacity. Animating layout properties (width, height, padding) forces layout every frame.
- Gesture-driven interactions belong in gesture-handler plus reanimated, not in PanResponder, which routes every move event over to JS.

## Startup

- Defer everything deferable: analytics init, remote config fetch, heavy SDK setup can start after first paint instead of before it.
- Watch top-level imports. A barrel file that imports the whole app to render the splash screen drags the entire dependency graph into startup. Import concretely and lazily require the rare heavy module.
- Fonts and lottie files add up; load what the first screen needs, not the full brand kit.
- Measure TTI on a mid-range Android device, cold start, release build. iPhone-simulator startup times are marketing numbers.

## JS thread stalls

- JSON.parse of megabyte payloads during interaction. Chunk it, move it behind InteractionManager, or trim the payload server-side.
- Synchronous storage reads in hot paths. Batch and cache; storage is IO wearing a synchronous costume.
- console.log in release builds costs real time on hot paths; strip or gate it.
- Date/number formatting in render for every row: precompute at data-massage time, once.

## When asked to "make it faster" with no specifics

Do this in order: turn on the performance monitor, reproduce the complaint, identify the busy thread, profile that thread, fix the top item only, re-measure, repeat. One measured fix at a time beats ten speculative ones, and the re-measure step is what keeps optimization honest.
