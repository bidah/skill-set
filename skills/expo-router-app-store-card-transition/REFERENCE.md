# Reference implementation

Distilled from `mobile-app/src/components/chat/discover.tsx`,
`mobile-app/src/lib/image-tint.ts`, and
`mobile-app/src/app/(tabs)/chat/discover/[slug].tsx` — read those for the
live versions with full commentary.

## Card

The poster: 16:10 picture over a caption block wearing the picture's own
colour. The whole card is `Link.AppleZoom`'s single child.

```tsx
import { Pressable, StyleSheet, Text, View } from 'react-native';
import { Link } from 'expo-router';
import { Image } from 'expo-image';
import * as Haptics from 'expo-haptics';
import { useImageTint } from '@/lib/image-tint';

const ART_RATIO = 16 / 10;

function PosterCard({ card }: { card: Card }) {
  // card.blurhash: build-time hash for shipped heroes; absent for remote media,
  // which gets hashed after its first download (blur-up from the second sight).
  const { blurhash, tint } = useImageTint(card.image, card.blurhash);

  return (
    <Link href={{ pathname: '/discover/[slug]', params: { slug: card.slug } }} asChild>
      <Pressable
        accessibilityRole="button"
        accessibilityLabel={`${card.title} — ${card.tagline}`}
        // Slot composes handlers: this runs ALONGSIDE the Link's navigation press.
        onPress={() => Haptics.selectionAsync().catch(() => {})}
        style={styles.shadow}
      >
        <Link.AppleZoom>
          {/* exactly ONE child — the whole card */}
          <View style={styles.card}>
            <Image
              source={card.image ? { uri: card.image } : undefined}
              style={styles.art}
              contentFit="cover"
              cachePolicy="memory-disk"
              // The picture's own blur at the same crop, so it blurs UP into
              // the real pixels instead of cutting.
              placeholder={{ blurhash: blurhash ?? FALLBACK_BLURHASH }}
              placeholderContentFit="cover"
              transition={{ duration: 420, effect: 'cross-dissolve' }}
            />
            {/* Falls back to theme colours until the tint is known. */}
            <View style={[styles.caption, { backgroundColor: tint?.background ?? themeSurface }]}>
              <Text style={[styles.kicker, { color: tint?.subtle ?? themeMuted }]} numberOfLines={1}>
                {card.kicker.toUpperCase()}
              </Text>
              <Text style={[styles.title, { color: tint?.foreground ?? themeFg }]} numberOfLines={2}>
                {card.title}
              </Text>
              <Text style={[styles.tagline, { color: tint?.subtle ?? themeMuted }]} numberOfLines={2}>
                {card.tagline}
              </Text>
            </View>
          </View>
        </Link.AppleZoom>
      </Pressable>
    </Link>
  );
}

const styles = StyleSheet.create({
  // Shadow on the OUTER wrapper: the card body clips (overflow hidden) to hold
  // its corner radius, and a clipping view drops its own shadow on iOS.
  shadow: {
    borderRadius: 24,
    shadowColor: '#000',
    shadowOpacity: 0.16,
    shadowRadius: 12,
    shadowOffset: { width: 0, height: 7 },
  },
  card: { borderRadius: 24, overflow: 'hidden' },
  art: { width: '100%', aspectRatio: ART_RATIO, backgroundColor: '#0b0b0c' },
  caption: { paddingHorizontal: 16, paddingTop: 12, paddingBottom: 14 },
  kicker: { fontSize: 11, fontWeight: '700', letterSpacing: 1.1 },
  title: { fontSize: 17, fontWeight: '700', lineHeight: 21, letterSpacing: -0.2, marginTop: 3 },
  tagline: { fontSize: 14, lineHeight: 18, marginTop: 2 },
});
```

## Tint

One blurhash gives placeholder + palette: chars 2–6 are the DC (average colour)
term, packed as three sRGB bytes — a string parse, not an image decode.

```ts
import AsyncStorage from '@react-native-async-storage/async-storage';
import { Image } from 'expo-image';

const BASE83 = '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz#$%*+,-.:;=?@[]^_{|}~';
const decode83 = (s: string) => [...s].reduce((v, c) => v * 83 + BASE83.indexOf(c), 0);

export function blurhashColor(hash?: string) {
  if (!hash || hash.length < 6) return null;
  const v = decode83(hash.slice(2, 6));
  return v < 0 ? null : { r: (v >> 16) & 255, g: (v >> 8) & 255, b: v & 255 };
}

// Raw photo averages are washed mid-tones: pull 22% toward the page background
// so the block reads as "this picture's colour" but belongs to the app. Then
// sRGB luma picks the text side.
export function tintFrom(rgb: RGB | null, pageBg: string) {
  if (!rgb) return null;
  const base = mix(rgb, parseHex(pageBg), 0.22);
  const light = (0.2126 * base.r + 0.7152 * base.g + 0.0722 * base.b) / 255 > 0.6;
  return {
    background: hex(base),
    foreground: light ? '#111114' : '#ffffff',
    subtle: light ? 'rgba(17,17,20,0.72)' : 'rgba(255,255,255,0.78)',
  };
}

// Hashing: 4×3 components — enough structure (horizon, subject, sky) without a
// long string. Cached in a module Map (session) + AsyncStorage (across
// launches), so the SECOND launch blurs-up before the download finishes.
const hash = await Image.generateBlurhashAsync(uri, [4, 3]);
```

Hook shape (`useImageTint(uri, seed?)`): `seed` is a build-time hash for
shipped images — with it, blur + tint exist on the FIRST frame, before a byte
of the picture arrives. Returns `{ blurhash, tint }`; consumers fall back to
theme colours while tint is null. Full version:
`mobile-app/src/lib/image-tint.ts`.

## Article

The destination. The hero is the same URI through the same hook, so the colour
is already cached when the zoom starts.

```tsx
export default function DetailScreen() {
  const { slug } = useLocalSearchParams<{ slug: string }>();
  const card = useEntry(slug);
  const { height } = useWindowDimensions();
  const { blurhash, tint } = useImageTint(card?.image, card?.blurhash);

  return (
    <View style={{ flex: 1 }}>
      {/* No bar title — the masthead below carries the headline. */}
      <Stack.Screen options={{ title: '' }} />
      {/* "never": the hero runs UNDER the transparent header so the bar's
          glass has a picture to refract. */}
      <ScrollView contentInsetAdjustmentBehavior="never">
        {/* The landing rect: the card's picture flies into THIS image. */}
        <Link.AppleZoomTarget>
          <Image
            source={card.image ? { uri: card.image } : undefined}
            style={{ width: '100%', height: Math.round(height * 0.62) - 170 }}
            contentFit="cover"
            cachePolicy="memory-disk"
            placeholder={{ blurhash: blurhash ?? FALLBACK_BLURHASH }}
            placeholderContentFit="cover"
            transition={{ duration: 420, effect: 'cross-dissolve' }}
          />
        </Link.AppleZoomTarget>

        {/* Masthead ONLY wears the image tint — same fill as the card's
            caption, which is what makes the zoom colour-continuous. */}
        <View style={{ backgroundColor: tint?.background ?? pageBg, padding: 20 }}>
          <Text style={[styles.kicker, { color: tint?.subtle ?? muted }]}>
            {card.kicker.toUpperCase()}
          </Text>
          <Text style={[styles.title, { color: tint?.foreground ?? fg }]}>{card.title}</Text>
          <Text style={[styles.standfirst, { color: tint?.subtle ?? muted }]}>{card.tagline}</Text>
        </View>

        {/* Body runs on the app's OWN colours — long copy on a photographic
            colour is illegible. Lead paragraph: bold opener via nested <Text>
            so both weights share one line box. */}
        <View style={{ backgroundColor: pageBg, paddingHorizontal: 20 }}>
          <Text style={styles.lead}>
            <Text style={styles.leadOpener}>{card.lead.opener}</Text>
            {card.lead.rest}
          </Text>
          {card.sections.map((s) => (
            <View key={s.heading}>
              <Text style={styles.heading}>{s.heading}</Text>
              <Text style={styles.copy}>{s.body}</Text>
            </View>
          ))}
        </View>
      </ScrollView>
    </View>
  );
}

// Editorial scale — a step up from UI type; this screen is something to READ.
const styles = StyleSheet.create({
  kicker: { fontSize: 12, fontWeight: '700', letterSpacing: 1.1 },
  title: { fontSize: 32, fontWeight: '700', lineHeight: 37, letterSpacing: -0.6 },
  standfirst: { fontSize: 17, lineHeight: 23 },
  lead: { fontSize: 19, lineHeight: 27 },
  leadOpener: { fontSize: 19, lineHeight: 27, fontWeight: '700' },
  heading: { fontSize: 22, fontWeight: '700', lineHeight: 27, marginTop: 30, marginBottom: 8 },
  copy: { fontSize: 18, lineHeight: 26 },
});
```
