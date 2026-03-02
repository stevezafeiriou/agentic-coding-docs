# State Management: TanStack Query + Zustand + MMKV

---

## Architecture Rule

```
Server state  → TanStack Query (with MMKV persister)
UI state      → Zustand (no persistence by default)
Persistent UI → Zustand + MMKV middleware
Local state   → useState / useReducer
URL/nav state → Expo Router search params
Form state    → React Hook Form
```

---

## TanStack Query: Setup with MMKV Persistence

```ts
// src/lib/query-client.ts
import { QueryClient } from "@tanstack/react-query";
import { createSyncStoragePersister } from "@tanstack/query-sync-storage-persister";
import { persistQueryClient } from "@tanstack/react-query-persist-client";
import { MMKV } from "react-native-mmkv";

const mmkv = new MMKV({ id: "tanstack-query-cache" });

// MMKV adapter for TanStack Query persister
const mmkvPersister = createSyncStoragePersister({
	storage: {
		getItem: (key) => mmkv.getString(key) ?? null,
		setItem: (key, value) => mmkv.set(key, value),
		removeItem: (key) => mmkv.delete(key),
	},
	// Debounce writes to avoid performance issues from rapid updates
	throttleTime: 1000,
});

export const queryClient = new QueryClient({
	defaultOptions: {
		queries: {
			staleTime: 1000 * 60 * 5, // 5 min — data is fresh
			gcTime: 1000 * 60 * 60, // 1 hour in memory
			retry: (count, error) => {
				if (error instanceof Error && error.message.includes("JWT"))
					return false;
				return count < 2;
			},
			networkMode: "offlineFirst", // Show cached data immediately
		},
		mutations: {
			networkMode: "offlineFirst",
		},
	},
});

// Persist cache to MMKV — survives app restarts
// Data older than 24h is discarded
persistQueryClient({
	queryClient,
	persister: mmkvPersister,
	maxAge: 1000 * 60 * 60 * 24, // 24 hours
	buster: "1", // Increment to bust cache on schema changes
});
```

---

## Query Key Factory

```ts
// src/lib/queries/keys.ts
export const queryKeys = {
	// Profile
	profile: {
		all: ["profile"] as const,
		current: () => [...queryKeys.profile.all, "current"] as const,
	},

	// Organizations
	orgs: {
		all: ["orgs"] as const,
		byUser: () => [...queryKeys.orgs.all, "user"] as const,
		detail: (id: string) => [...queryKeys.orgs.all, id] as const,
		members: (id: string) => [...queryKeys.orgs.all, id, "members"] as const,
	},

	// Projects
	projects: {
		all: ["projects"] as const,
		byOrg: (orgId: string) =>
			[...queryKeys.projects.all, "org", orgId] as const,
		detail: (id: string) => [...queryKeys.projects.all, id] as const,
	},

	// Tasks
	tasks: {
		all: ["tasks"] as const,
		byProject: (projectId: string) =>
			[...queryKeys.tasks.all, "project", projectId] as const,
		detail: (id: string) => [...queryKeys.tasks.all, id] as const,
	},
} as const;
```

---

## Data Hook with Optimistic Mutation

```ts
// src/hooks/use-tasks.ts
import {
	useQuery,
	useMutation,
	useQueryClient,
	useInfiniteQuery,
} from "@tanstack/react-query";
import {
	getTasksByProject,
	getTasksPaginated,
	type Task,
} from "@/lib/queries/tasks";
import { createTask, updateTask, deleteTask } from "@/lib/mutations/tasks";
import { queryKeys } from "@/lib/queries/keys";
import { toast } from "@/utils/toast";

export function useTasks(projectId: string) {
	return useQuery({
		queryKey: queryKeys.tasks.byProject(projectId),
		queryFn: () => getTasksByProject(projectId),
		enabled: !!projectId,
	});
}

// Infinite scroll for large lists
export function useInfiniteTasks(projectId: string) {
	return useInfiniteQuery({
		queryKey: [...queryKeys.tasks.byProject(projectId), "infinite"],
		queryFn: ({ pageParam = 0 }) => getTasksPaginated(projectId, pageParam),
		initialPageParam: 0,
		getNextPageParam: (lastPage, _, lastPageParam) =>
			lastPage.hasMore ? lastPageParam + 1 : undefined,
	});
}

export function useCreateTask() {
	const queryClient = useQueryClient();

	return useMutation({
		mutationFn: createTask,

		onMutate: async (input) => {
			// Cancel in-flight refetches
			await queryClient.cancelQueries({
				queryKey: queryKeys.tasks.byProject(input.project_id),
			});
			const previous = queryClient.getQueryData(
				queryKeys.tasks.byProject(input.project_id),
			);

			// Optimistic insert with temp id
			const optimisticTask: Task = {
				id: `optimistic-${Date.now()}`,
				created_at: new Date().toISOString(),
				is_complete: false,
				...input,
			};

			queryClient.setQueryData(
				queryKeys.tasks.byProject(input.project_id),
				(old: Task[] = []) => [optimisticTask, ...old],
			);

			return { previous };
		},

		onError: (err, input, context) => {
			queryClient.setQueryData(
				queryKeys.tasks.byProject(input.project_id),
				context?.previous,
			);
			toast.error("Failed to create task");
		},

		onSettled: (_, __, input) => {
			queryClient.invalidateQueries({
				queryKey: queryKeys.tasks.byProject(input.project_id),
			});
		},
	});
}

export function useDeleteTask(projectId: string) {
	const queryClient = useQueryClient();

	return useMutation({
		mutationFn: deleteTask,

		onMutate: async (taskId) => {
			await queryClient.cancelQueries({
				queryKey: queryKeys.tasks.byProject(projectId),
			});
			const previous = queryClient.getQueryData(
				queryKeys.tasks.byProject(projectId),
			);

			queryClient.setQueryData(
				queryKeys.tasks.byProject(projectId),
				(old: Task[] = []) => old.filter((t) => t.id !== taskId),
			);

			return { previous };
		},

		onError: (_, __, context) => {
			queryClient.setQueryData(
				queryKeys.tasks.byProject(projectId),
				context?.previous,
			);
			toast.error("Failed to delete task");
		},

		onSettled: () => {
			queryClient.invalidateQueries({
				queryKey: queryKeys.tasks.byProject(projectId),
			});
		},
	});
}
```

---

## Zustand: UI State Only

```ts
// src/stores/app-store.ts
import { create } from "zustand";
import { createJSONStorage, persist } from "zustand/middleware";
import { MMKV } from "react-native-mmkv";

const mmkv = new MMKV({ id: "app-store" });
const mmkvStorage = {
	getItem: (name: string) => mmkv.getString(name) ?? null,
	setItem: (name: string, value: string) => mmkv.set(name, value),
	removeItem: (name: string) => mmkv.delete(name),
};

interface AppState {
	// Active org — persisted
	activeOrgId: string | null;
	setActiveOrgId: (id: string) => void;

	// Notification badge — not persisted
	unreadCount: number;
	setUnreadCount: (count: number) => void;
	incrementUnread: () => void;
	clearUnread: () => void;
}

export const useAppStore = create<AppState>()(
	persist(
		(set) => ({
			activeOrgId: null,
			setActiveOrgId: (id) => set({ activeOrgId: id }),

			unreadCount: 0,
			setUnreadCount: (count) => set({ unreadCount: count }),
			incrementUnread: () => set((s) => ({ unreadCount: s.unreadCount + 1 })),
			clearUnread: () => set({ unreadCount: 0 }),
		}),
		{
			name: "app-state",
			storage: createJSONStorage(() => mmkvStorage),
			// Only persist specific keys
			partialize: (state) => ({ activeOrgId: state.activeOrgId }),
		},
	),
);
```

---

## Offline Mutation Queue

For mutations that should retry when connectivity returns:

```ts
// src/lib/offline-queue.ts
import { MMKV } from "react-native-mmkv";
import NetInfo from "@react-native-community/netinfo";

const storage = new MMKV({ id: "offline-queue" });
const QUEUE_KEY = "mutation-queue";

interface QueuedMutation {
	id: string;
	type: string;
	payload: unknown;
	createdAt: string;
}

export function enqueueOfflineMutation(type: string, payload: unknown) {
	const existing: QueuedMutation[] = JSON.parse(
		storage.getString(QUEUE_KEY) ?? "[]",
	);
	const mutation: QueuedMutation = {
		id: `${Date.now()}-${Math.random()}`,
		type,
		payload,
		createdAt: new Date().toISOString(),
	};
	storage.set(QUEUE_KEY, JSON.stringify([...existing, mutation]));
}

export function getQueuedMutations(): QueuedMutation[] {
	return JSON.parse(storage.getString(QUEUE_KEY) ?? "[]");
}

export function clearQueue() {
	storage.delete(QUEUE_KEY);
}

// Call on app startup and when connectivity returns
export async function processOfflineQueue(
	handlers: Record<string, (payload: unknown) => Promise<void>>,
) {
	const state = await NetInfo.fetch();
	if (!state.isConnected) return;

	const queue = getQueuedMutations();
	if (queue.length === 0) return;

	const failed: QueuedMutation[] = [];

	for (const mutation of queue) {
		const handler = handlers[mutation.type];
		if (!handler) continue;

		try {
			await handler(mutation.payload);
		} catch {
			failed.push(mutation);
		}
	}

	storage.set(QUEUE_KEY, JSON.stringify(failed));
}
```
