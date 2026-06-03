---
name: react-native-keyboard-handling
description: >-
  Implement frame-perfect, native-feeling keyboard handling in React Native and
  Expo apps. Use when content is covered by the keyboard, when keyboard
  animations jank or "snap" on Android, when inputs are hidden behind the
  keyboard, when the keyboard covers fields on Android 15 / edge-to-edge even
  though older Android worked, or when building forms, sticky footers, or chat
  input bars that must track the keyboard. Triggers: "keyboard covers input",
  "KeyboardAvoidingView not working on Android", "keyboard animation janky",
  "keyboard sticky footer", "Android 15 keyboard", "edge-to-edge keyboard",
  react-native-keyboard-controller.
---

# React Native Keyboard Handling

Make the keyboard feel native on both platforms by mapping two fundamentally
different OS keyboard models onto **one animated value** that your UI binds to.

## The core mental model (read this first)

iOS and Android expose the keyboard in incompatible ways. Most "broken keyboard"
bugs come from ignoring this asymmetry.

- **iOS = scheduled animation.** The OS tells you up front: "the keyboard will
  slide from A to B over 0.25s with this easing curve." You get two snapshots
  (start/end) plus duration and curve — *not* intermediate frames. UIKit
  interpolates for you.
- **Android (modern, edge-to-edge) = per-frame insets.** The IME is just another
  system window. You get a firehose of frames, one inset value per frame, and
  you drive the animation yourself.

> Anything that ignores this asymmetry will feel broken. The fix is to map both
> models onto a single shared animated value, then have every consumer (padded
> container, translated footer, scroll offset) bind to that one value and stop
> caring which platform feeds it.

## Decision guide — which tool to reach for

| Situation | Use |
|---|---|
| Whole screen / single input should stay above keyboard | `KeyboardAvoidingView` (from `react-native-keyboard-controller`) |
| Scrollable form with multiple inputs | `KeyboardAwareScrollView` |
| One element pinned above the keyboard (footer, send bar, toolbar) | `KeyboardStickyView` |
| Fully custom animation tied to keyboard position | `useReanimatedKeyboardAnimation()` / `useKeyboardHandler()` + Reanimated `SharedValue` |

**Default recommendation:** use
[`react-native-keyboard-controller`](https://github.com/kirillzyusko/react-native-keyboard-controller),
not React Native's built-in `KeyboardAvoidingView`. The built-in version relies
on `LayoutAnimation` and the late `keyboardDidShow` event, which has no per-frame
Android primitive — so Android content snaps instead of sliding and the timing
never matches the keyboard curve.

## Setup

1. Install the library and its peer deps:
   ```bash
   npm install react-native-keyboard-controller react-native-reanimated
   ```
2. Wrap your app root **once** in `KeyboardProvider`. This installs the native
   `WindowInsetsAnimationCallback` subscription on Android and handles
   edge-to-edge automatically:
   ```tsx
   import { KeyboardProvider } from "react-native-keyboard-controller";

   export default function App() {
     return (
       <KeyboardProvider>
         {/* ...rest of the app */}
       </KeyboardProvider>
     );
   }
   ```

## Patterns

### Single input / simple screen — drop-in replacement
Same API as RN's `KeyboardAvoidingView`; swapping the import alone fixes the
Android timing/snap problems.
```tsx
import { KeyboardAvoidingView } from "react-native-keyboard-controller";

<KeyboardAvoidingView behavior="padding" style={{ flex: 1 }}>
  {/* input(s) */}
</KeyboardAvoidingView>
```

### Scrollable form — track the focused input
Auto-scrolls the minimum distance to keep the focused input (and caret/selection)
above the keyboard, using the same curve as the keyboard motion. `bottomOffset`
adds breathing room between the input and the keyboard top.
```tsx
import { KeyboardAwareScrollView } from "react-native-keyboard-controller";

<KeyboardAwareScrollView bottomOffset={50} style={{ flex: 1 }}>
  <TextInput placeholder="Field 1" />
  <TextInput placeholder="Field 2" />
  {/* more inputs */}
</KeyboardAwareScrollView>
```

### Sticky footer / send bar — translate one element only
Translates a single element instead of recomputing flex layout for the whole
screen. `offset` controls the gap when the keyboard is open vs closed.
```tsx
import { KeyboardStickyView } from "react-native-keyboard-controller";

<KeyboardStickyView offset={{ closed: 0, opened: 20 }}>
  <Footer />
</KeyboardStickyView>
```

### Custom animation — bind to the shared keyboard value
For fully bespoke motion, read the keyboard height as a Reanimated `SharedValue`
and drive any style from it. Works identically on both platforms because the
library already merged the two OS models into this one value.
```tsx
import { useReanimatedKeyboardAnimation } from "react-native-keyboard-controller";
import Animated, { useAnimatedStyle } from "react-native-reanimated";

const { height } = useReanimatedKeyboardAnimation(); // SharedValue, per-frame
const style = useAnimatedStyle(() => ({
  transform: [{ translateY: height.value }],
}));

<Animated.View style={style}>{/* ... */}</Animated.View>;
```

## Pitfalls & gotchas

- **Android 15 / SDK 35 covers your inputs.** Edge-to-edge is forced by default
  from Android 15 (API 35). The system no longer resizes the window when the
  keyboard appears, so the old `adjustResize` / `windowSoftInputMode` contract
  silently breaks — inputs get covered even though the exact same code worked on
  Android 14. `KeyboardProvider` handles edge-to-edge for you; don't rely on
  `adjustResize`.
- **`will*` events never fire on Android.** `keyboardWillShow` / `keyboardWillHide`
  have no underlying Android primitive, and `keyboardDidShow` fires *after* the
  animation finishes — too late to animate in sync. Don't build animations on
  the raw `Keyboard` event listeners for Android.
- **Stock `KeyboardAvoidingView` + Android = snap, not slide.** It uses
  `LayoutAnimation`, whose timing doesn't match the keyboard curve on Android, so
  motion looks janky. Prefer the keyboard-controller version.
- **Version fragmentation is real.** Behavior that "works" on Android 10 can break
  differently on Android 11 and again on Android 15. Use a library that abstracts
  the OS differences instead of hand-rolling per-version logic.
- **Wrong component for the job.** Using `KeyboardAvoidingView` for a multi-input
  form won't track *which* input is focused, so the caret can stay hidden — use
  `KeyboardAwareScrollView`. Using it to move a single footer forces an
  unnecessary full-screen layout recompute — use `KeyboardStickyView`.

## Why the asymmetry exists (native reference)

Useful when debugging at the native layer or explaining the design.

- **iOS surface:** `NotificationCenter` keyboard notifications carry
  `keyboardFrameEndUserInfoKey` (final frame), `keyboardAnimationDurationUserInfoKey`
  (timing), and `keyboardAnimationCurveUserInfoKey` (a private curve,
  `UIView.AnimationCurve(rawValue: 7)`). Two snapshots, not frames.
- **Android surface:** `WindowInsetsAnimationCallback` with four hooks —
  `onPrepare()`, `onStart(insets)`, `onProgress(insets, runningAnimations)` (fires
  every frame), `onEnd()`. The library subscribes here and writes each frame's
  inset into the shared value; on Android 11 it polyfills the per-frame stream
  where native support is missing.

## Source

Based on Margelo's *The Go-To Guide for Understanding Keyboards in React Native (Part 1)*:
https://blog.margelo.com/deep-dive-in-keyboard-handling
