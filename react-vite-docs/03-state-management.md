# State Management Architecture

---

## The Golden Rule

**Server state and UI state are different problems.**
Never mix them.

| State Type      | Tool                   | Examples                                          |
| --------------- | ---------------------- | ------------------------------------------------- |
| Server state    | TanStack Query         | Users, projects, invoices, anything from Supabase |
| Global UI state | Zustand                | Sidebar open/closed, active org, theme, toasts    |
| Local UI state  | useState / useReducer  | Form values, hover states, modal open             |
| URL state       | Search params / router | Filters, pagination, selected tab                 |
| Form state      | React Hook Form        | Input values, validation, dirty state             |

---

## TanStack Query: Server State

### Setup

```tsx
// src/main.tsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";

const queryClient = new QueryClient({
	defaultOptions: {
		queries: {
			staleTime: 1000 * 60 * 5, // 5 minutes — data is fresh
			gcTime: 1000 * 60 * 10, // 10 minutes — keep in cache
			retry: (failureCount, error) => {
				// Don't retry on 4xx errors
				if (error instanceof Error && error.message.includes("403"))
					return false;
				return failureCount < 2;
			},
		},
		mutations: {
			onError: (error) => {
				console.error("Mutation error:", error);
			},
		},
	},
});

export function App() {
	return (
		<QueryClientProvider client={queryClient}>
			<YourApp />
			{import.meta.env.DEV && <ReactQueryDevtools />}
		</QueryClientProvider>
	);
}
```

### Query Key Factory Pattern

```ts
// src/lib/queries/keys.ts
// Centralize all query keys to prevent typos and enable precise invalidation

export const queryKeys = {
	// Organizations
	orgs: {
		all: ["orgs"] as const,
		byUser: () => [...queryKeys.orgs.all, "by-user"] as const,
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

## Zustand: Global UI State

**Rule:** Zustand stores contain ZERO server data. If you're tempted to store
fetched data in Zustand, use TanStack Query instead.

### Organization Store (Multi-Tenant)

```ts
// src/stores/org-store.ts
import { create } from "zustand";
import { persist } from "zustand/middleware";

interface OrgState {
	activeOrgId: string | null;
	setActiveOrg: (orgId: string) => void;
}

export const useOrgStore = create<OrgState>()(
	persist(
		(set) => ({
			activeOrgId: null,
			setActiveOrg: (orgId) => set({ activeOrgId: orgId }),
		}),
		{ name: "active-org" }, // Persists to localStorage
	),
);

// Convenience hook
export function useActiveOrgId(): string {
	const id = useOrgStore((s) => s.activeOrgId);
	if (!id) throw new Error("No active org — redirect to org selection");
	return id;
}
```

### UI Store

```ts
// src/stores/ui-store.ts
import { create } from "zustand";

interface UIState {
	sidebarOpen: boolean;
	setSidebarOpen: (open: boolean) => void;
	toggleSidebar: () => void;
}

export const useUIStore = create<UIState>((set) => ({
	sidebarOpen: true,
	setSidebarOpen: (open) => set({ sidebarOpen: open }),
	toggleSidebar: () => set((s) => ({ sidebarOpen: !s.sidebarOpen })),
}));
```

---

## URL State: Filters & Pagination

Keep filter/pagination state in the URL so it's shareable and survives refresh.

```tsx
// src/hooks/use-project-filters.ts
import { useSearchParams } from "react-router-dom";

interface ProjectFilters {
	status: string | null;
	search: string;
	page: number;
}

export function useProjectFilters() {
	const [params, setParams] = useSearchParams();

	const filters: ProjectFilters = {
		status: params.get("status"),
		search: params.get("search") ?? "",
		page: parseInt(params.get("page") ?? "1", 10),
	};

	function setFilter<K extends keyof ProjectFilters>(
		key: K,
		value: ProjectFilters[K],
	) {
		setParams(
			(prev) => {
				const next = new URLSearchParams(prev);
				if (value === null || value === "" || value === 1) {
					next.delete(key);
				} else {
					next.set(key, String(value));
				}
				// Reset to page 1 when filters change
				if (key !== "page") next.delete("page");
				return next;
			},
			{ replace: true },
		);
	}

	return { filters, setFilter };
}
```
