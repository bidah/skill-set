---
name: expo-router-app-store-card-transition
description: Build an App Store "Today"-style card that zooms into its own detail/article screen with expo-router's Link.AppleZoom, including the poster card UI (expo-image + blurhash placeholder, caption tinted with the image's own average colour) and the editorial destination screen. Use when adding a card-to-detail zoom transition, an App Store-like poster card, blurhash tinting, Link.AppleZoom / Link.AppleZoomTarget, or a Today-style article screen in an Expo app.
---

# App Store card → article zoom (expo-router)

Three parts that only work as a set: a **poster card**, the **zoom transition**,
and the **article screen** it lands on. Working reference implementation:
`mobile-app/src/components/chat/discover.tsx` (card) and
`mobile-app/src/app/(tabs)/chat/discover/[slug].tsx` (destination), with the
tint library in `mobile-app/src/lib/image-tint.ts`.

## Requirements

- **expo-router above v55** (this repo runs `~57.0.4`, SDK 57) — `Link.AppleZoom`
  / `Link.AppleZoomTarget` don't exist in older routers.
- **expo-image** — supplies the blurhash placeholder AND
  `Image.generateBlurhashAsync` for the tint.
- **iOS 18+** for the zoom itself. Off iOS (and on older iOS) the components are
  a passthrough: the press degrades to a plain stack push. Ship it anyway —
  nothing crashes, Android just pushes.

## The transition — rules that are load-bearing

```tsx
<Link href={{ pathname: '/discover/[slug]', params: { slug } }} asChild>
  <Pressable onPress={() => Haptics.selectionAsync().catch(() => {})}>
    <Link.AppleZoom>
      <View>{/* the WHOLE card — image + caption — as ONE child */}</View>
    </Link.AppleZoom>
  </Pressable>
</Link>
```

1. **`<Link asChild>` is required, not stylistic.** The router threads the zoom
   through the HREF (it rewrites it with a source id the destination reads off
   its route params). `router.push` gets no zoom at all, and `Link.AppleZoom`
   **throws** without `asChild`.
2. **`Link.AppleZoom` takes exactly ONE child** — more than one warns and
   renders nothing. Wrap the whole card in a single view; the card is what
   expands.
3. **Destination marks the landing rect** with `<Link.AppleZoomTarget>` around
   its hero image — otherwise the card expands into the screen as a whole
   instead of the picture flying into place.
4. **No press-scale wrapper on the card** (no BouncyPressable): a press-shrink
   transform fights the zoom for the same frame — the zoom IS the feedback.
   A plain `Pressable` with a selection haptic is enough. (Radix Slot composes
   handlers, so `onPress` runs alongside the Link's navigation press.)
5. **Build the card as a small copy of the destination** — picture on top, text
   block below in the picture's own colour — so the zoom has something to land
   on and the colour continues across the transition.

## The card (poster)

`expo-image` with the picture's own blurhash as placeholder (same crop, so one
blurs UP into the other), `cross-dissolve` transition, caption block filled with
the image's tint. Shadow on an outer wrapper; the card body uses
`overflow: 'hidden'` for its corner radius (a clipping view drops its own shadow
on iOS). Full code: [REFERENCE.md](REFERENCE.md#card).

## The tint — one artefact, two jobs

A blurhash's first component (the DC term, base83 chars 2–6) **is the image's
average sRGB colour** — so one `Image.generateBlurhashAsync(uri, [4, 3])` yields
the placeholder AND the palette. No native palette extractor, no second decode.

- Parse the DC term with a ~20-line base83 decode ([REFERENCE.md](REFERENCE.md#tint)).
- Mix the raw average **22% toward the page background** (raw photo averages are
  washed mid-tones), then pick white / near-black text by sRGB luma > 0.6.
- Cache hashes in a module `Map` + AsyncStorage so the second launch blurs-up
  before the image downloads. A **first** sight can't be blurred (hashing needs
  pixels) — ship a build-time hash (`seed`) for bundled/known heroes.

## The article screen

- Hero runs **under the transparent header**: `contentInsetAdjustmentBehavior="never"`
  on the ScrollView, `Stack.Screen options={{ title: '' }}` (the masthead below
  carries the headline; the bar keeps only chevron + menu).
- Hero wrapped in `Link.AppleZoomTarget`, same URI + same `useImageTint` call as
  the card — the hash is already in the session cache, so the colour is there on
  the first frame of the zoom.
- **Tint the masthead only** (kicker / title / standfirst). Body copy runs on the
  app's own page colours — three screens of text on an arbitrary photographic
  colour is illegible.
- Editorial type scale, one step up from UI type; lead paragraph opens with a
  bold foreground-coloured phrase via a **nested `<Text>`** so both weights share
  one flowing line box. Full code: [REFERENCE.md](REFERENCE.md#article).
