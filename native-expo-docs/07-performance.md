# Performance: Caching, Bundle Size, Profiling

---

## The Mobile Performance Contract

1. **App launch → interactive in < 2 seconds** (cold start)
2. **Cached screens render in < 100ms** (MMKV + TanStack Query)
3. **List scroll at 60fps** (FlashList + memoized items)
4. **No JS thread blocking** (Reanimated for animations)
5. **No unnecessary re-renders** (stable references, memoization where profiled)

---

## Supabase: Minimize Requests

### Column Selection — Only Fetch What You Display

```ts
// BAD: fetches entire row including blobs/long text
const { data } = await supabase.from("projects").select("*");

// GOOD: fetch only what the list renders
const { data } = await supabase
	.from("projects")
	.select("id, name, status, member_count, updated_at");
```

### Query Batching — Single Request for Related Data

```ts
// BAD: two separate round-trips
const project = await getProject(id);
const members = await getProjectMembers(id);

// GOOD: single query with join
const { data } = await supabase
	.from("projects")
	.select(
		`
    *,
    members:project_members(
      user:profiles(id, full_name, avatar_url)
    )
  `,
	)
	.eq("id", id)
	.single();
```

### Avoid N+1 Queries

```ts
// BAD: one query per task to get assignee
const tasks = await getTasks(projectId);
const withAssignees = await Promise.all(
	tasks.map((t) => getUser(t.assignee_id)), // N+1!
);

// GOOD: join in single query
const { data } = await supabase
	.from("tasks")
	.select("*, assignee:profiles(id, full_name, avatar_url)")
	.eq("project_id", projectId);
```

### Pagination — Never Unbounded Queries

```ts
// BAD: potentially thousands of rows
const { data } = await supabase.from("activities").select("*");

// GOOD: always limit + paginate
const { data } = await supabase
	.from("activities")
	.select("*")
	.order("created_at", { ascending: false })
	.limit(20) // Hard limit
	.range(page * 20, (page + 1) * 20 - 1); // Cursor pagination
```

---

## TanStack Query: Cache Strategy by Data Type

```ts
const STALE_TIMES = {
	// User's own profile — can be slightly stale
	profile: 1000 * 60 * 10, // 10 minutes

	// Org data — changes infrequently
	orgDetails: 1000 * 60 * 30, // 30 minutes

	// Lists — should feel fresh
	taskLists: 1000 * 60 * 5, // 5 minutes

	// Real-time augmented by subscription — can be stalest
	notifications: 1000 * 30, // 30 seconds

	// Reference data — almost never changes
	orgMembers: 1000 * 60 * 15, // 15 minutes
};
```

### Prefetch on Hover/Focus

```ts
// Prefetch detail when user focuses a list item
// In a list, trigger prefetch on visible item
function TaskRow({ task }: { task: Task }) {
	const queryClient = useQueryClient();

	return (
		<Pressable
			onPress={() => router.push(`/tasks/${task.id}`)}
			onPressIn={() => {
				// Prefetch on press-in — data ready by the time navigation completes
				queryClient.prefetchQuery({
					queryKey: queryKeys.tasks.detail(task.id),
					queryFn: () => getTaskDetail(task.id),
					staleTime: 1000 * 60,
				});
			}}
		>
			...
		</Pressable>
	);
}
```

---

## FlashList Performance Rules

```tsx
// Non-negotiable FlashList rules:

// 1. estimatedItemSize must be accurate — measure your items
<FlashList estimatedItemSize={80} />

// 2. renderItem MUST be stable
const renderItem = useCallback(({ item }) => <TaskCard task={item} />, [])

// 3. keyExtractor MUST be stable
const keyExtractor = useCallback((item: Task) => item.id, [])

// 4. Item components should be wrapped in React.memo
export const TaskCard = React.memo(function TaskCard({ task }: { task: Task }) {
  // ... only re-renders when task reference changes
})

// 5. Use getItemType for heterogeneous lists
const getItemType = useCallback((item: FeedItem) => item.type, [])
<FlashList getItemType={getItemType} />
```

---

## Image Performance (expo-image)

```tsx
import { Image } from "expo-image";

// expo-image caches aggressively on disk by default
// Never use RN's built-in Image for remote content

export function Avatar({
	url,
	size = 40,
}: {
	url: string | null;
	size?: number;
}) {
	return (
		<Image
			source={url ?? require("@/assets/default-avatar.png")}
			style={{ width: size, height: size, borderRadius: size / 2 }}
			contentFit="cover"
			transition={200} // Smooth fade-in
			cachePolicy="memory-disk" // Default — keeps in disk cache
			recyclingKey={url ?? "default"} // Re-use in FlashList
		/>
	);
}
```

---

## React.memo — When and How

```tsx
// WHEN: component is in a FlashList, receives stable props, renders often
export const NotificationRow = React.memo(
  function NotificationRow({ notification, onRead }: Props) {
    return (...)
  },
  // Custom comparison: only re-render if is_read changes
  (prev, next) =>
    prev.notification.id      === next.notification.id &&
    prev.notification.is_read === next.notification.is_read &&
    prev.onRead               === next.onRead
)

// WHEN NOT: simple components that render rarely
// Bad: wrapping everything in memo adds overhead without benefit
export const Label = React.memo(({ text }: { text: string }) => (
  <Text>{text}</Text>
)) // ❌ Not worth it — Label is trivial
```

---

## Preventing Re-Renders: Selector Pattern

```ts
// BAD: any store change re-renders this component
function Header() {
	const store = useAppStore(); // Re-renders on ANY state change
	return <Text>{store.activeOrgId}</Text>;
}

// GOOD: component only re-renders when activeOrgId changes
function Header() {
	const activeOrgId = useAppStore((s) => s.activeOrgId);
	return <Text>{activeOrgId}</Text>;
}
```

---

## Bundle Size: Reducing Cold Start

```ts
// babel.config.js — tree-shake lodash, date-fns, etc.
module.exports = {
	plugins: [
		[
			"babel-plugin-transform-imports",
			{
				"date-fns": {
					transform: "date-fns/${member}",
					preventFullImport: true,
				},
			},
		],
	],
};
```

```ts
// Use metro bundle analyzer to find large modules:
// npx expo export --platform ios && npx react-native-bundle-visualizer

// Common culprits:
// - moment.js → replace with date-fns
// - lodash → import specific functions only
// - @supabase/supabase-js → already tree-shakeable
```

---

## Profiling Checklist

Run these checks before shipping any screen:

```
□ Flipper / React DevTools Profiler: no component renders > 16ms
□ FlashList: all items render in a single frame when scrolling fast
□ Network tab: no duplicate Supabase requests for same data
□ Network tab: no unbounded queries (missing .limit())
□ Memory: no subscription leaks (check on screen unmount)
□ Startup: MMKV cache hit on second launch (data shows without loading spinner)
□ Offline: cached screens show data with "offline" indicator, not blank
□ Mutations: optimistic update shows before network completes
□ Images: all images sized to display size (no 2x over-fetching)
□ No console warnings about missing keys, deprecated APIs
```

---

## AppState + NetInfo Integration

```ts
// src/lib/app-lifecycle.ts
// Handle background/foreground and connectivity changes

import { AppState } from "react-native";
import NetInfo from "@react-native-community/netinfo";
import { queryClient } from "./query-client";
import { processOfflineQueue } from "./offline-queue";

export function setupAppLifecycle() {
	// Re-focus stale queries when app returns to foreground
	const appStateListener = AppState.addEventListener("change", (state) => {
		if (state === "active") {
			queryClient.resumePausedMutations();
			queryClient.invalidateQueries(); // Mark all as potentially stale
		}
	});

	// Process offline queue when connectivity returns
	const netInfoListener = NetInfo.addEventListener((state) => {
		if (state.isConnected) {
			queryClient.resumePausedMutations();
			processOfflineQueue(mutationHandlers);
		}
	});

	return () => {
		appStateListener.remove();
		netInfoListener();
	};
}
```
