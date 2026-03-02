# Core Principles & Decision Checklist

# React Native + Expo + NativeWind + Supabase

> Every AI agent must read this file before writing any code.

---

## Non-Negotiable Principles

**TypeScript strict mode everywhere.**
No `any`. No `as unknown as X`. No implicit `any` in function parameters.
Generated Supabase types flow through the entire app — never re-declare
types that already exist in the database schema.

**Never call Supabase directly from a component.**
All Supabase interactions live in `/lib/queries/` (reads) and
`/lib/mutations/` (writes). Components call hooks. Hooks call query
functions. This makes caching, testing, and refactoring possible.

**Cache aggressively, fetch sparingly.**
Mobile networks are unreliable and expensive. Every Supabase query goes
through TanStack Query. MMKV persists critical data across app restarts.
The user should never see a loading spinner for data they've already seen.

**Optimistic updates for all mutations.**
The user's action must be reflected instantly. Roll back on error with a
visible toast. Never make a user wait for a round-trip to see their own
action reflected.

**RLS is the only security layer.**
Frontend role checks are UX only — never security. If a Supabase RLS
policy doesn't exist for an operation, that operation cannot happen,
regardless of what the frontend does.

**Platform awareness is required.**
iOS and Android behave differently for notifications, keyboard, safe areas,
shadows, fonts, and gestures. Every component that differs by platform
uses `Platform.OS` explicitly — never assume iOS behavior on Android.

**Offline-first mindset.**
Mobile users go offline constantly. Every screen that shows data must handle
the "no network, cached data available" state gracefully. Mutations queue
when offline and retry when connectivity returns.

---

## Stack Reference

| Concern       | Tool                         | Version / Notes                       |
| ------------- | ---------------------------- | ------------------------------------- |
| Framework     | React Native + Expo SDK      | SDK 51+                               |
| Language      | TypeScript                   | Strict mode                           |
| Routing       | Expo Router v3               | File-based, typed                     |
| Styling       | NativeWind v4                | Tailwind CSS for RN                   |
| Backend       | Supabase                     | Postgres, Auth, Realtime, Storage     |
| Server state  | TanStack Query v5            | With MMKV persister                   |
| Client state  | Zustand                      | UI state only                         |
| Persistence   | MMKV (react-native-mmkv)     | Fast key-value, replaces AsyncStorage |
| Forms         | React Hook Form + Zod        |                                       |
| Notifications | Expo Notifications           | With expo-device                      |
| Gestures      | React Native Gesture Handler |                                       |
| Animations    | React Native Reanimated v3   |                                       |
| Lists         | FlashList (Shopify)          | Replaces FlatList everywhere          |
| Images        | Expo Image                   | Replaces Image everywhere             |
| Icons         | @expo/vector-icons           | Ionicons / MaterialIcons              |
| Date          | date-fns                     |                                       |

---

## Project Structure

```
app/                          # Expo Router file-based routes
├── (auth)/
│   ├── _layout.tsx           # Auth stack layout
│   ├── login.tsx
│   └── register.tsx
├── (app)/
│   ├── _layout.tsx           # Tab/drawer layout (protected)
│   ├── (tabs)/
│   │   ├── index.tsx         # Home tab
│   │   ├── explore.tsx
│   │   └── profile.tsx
│   └── [entity]/
│       └── [id].tsx          # Detail screens
├── _layout.tsx               # Root layout (providers)
└── +not-found.tsx

src/
├── components/
│   ├── ui/                   # Primitives: Button, Card, Input, Badge
│   └── [feature]/            # Feature-specific components
├── hooks/                    # Custom hooks
├── lib/
│   ├── supabase/
│   │   ├── client.ts         # Supabase client (singleton)
│   │   ├── types.ts          # Re-export generated types
│   │   └── auth-context.tsx  # Auth provider
│   ├── queries/              # All SELECT functions
│   ├── mutations/            # All INSERT/UPDATE/DELETE functions
│   └── validations/          # Zod schemas
├── stores/                   # Zustand stores
├── types/                    # Shared TS types
└── utils/                    # Pure utilities (cn, format, etc.)
```

---

## New Feature Checklist

### Before writing any code:

- [ ] Which Supabase table(s) does this touch?
- [ ] Are RLS policies written for all operations (SELECT, INSERT, UPDATE, DELETE)?
- [ ] Is there a generated type for this table? Run `npm run db:types` if not.
- [ ] Does this data need to persist across app restarts? → MMKV persister
- [ ] Does this need real-time updates? → Supabase Realtime channel
- [ ] What happens offline? → Define the offline state explicitly

### Component checklist:

- [ ] Does this render a list? → FlashList, not FlatList/ScrollView+map
- [ ] Does this show an image? → expo-image, not Image
- [ ] Does this need safe area insets? → `useSafeAreaInsets()` or SafeAreaView
- [ ] Does this handle keyboard? → KeyboardAvoidingView with platform behavior
- [ ] Does this differ on iOS vs Android? → explicit Platform.OS check
- [ ] Does this have loading / error / empty states?
- [ ] Is tap target at least 44×44pt? (Apple HIG minimum)

### Performance checklist:

- [ ] Are list items memoized with useCallback for keyExtractor/renderItem?
- [ ] Are expensive computations behind useMemo?
- [ ] Are images sized correctly? Never larger than displayed size.
- [ ] Does this component need to be on the JS thread? Can Reanimated handle it?
