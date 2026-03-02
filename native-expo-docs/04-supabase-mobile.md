# Supabase for Mobile: Auth, Queries, Caching, Offline

---

## Client Setup (Mobile-Specific)

Mobile Supabase clients use `AsyncStorage` or — preferably — MMKV
for session persistence. Session tokens survive app restarts.

```ts
// src/lib/supabase/client.ts
import { createClient } from "@supabase/supabase-js";
import { MMKV } from "react-native-mmkv";
import type { Database } from "./types";
import { env } from "@/lib/env";

// MMKV is synchronous and 10x faster than AsyncStorage
const storage = new MMKV({ id: "supabase-session" });

const mmkvStorageAdapter = {
	getItem: (key: string) => {
		const value = storage.getString(key);
		return value ?? null;
	},
	setItem: (key: string, value: string) => {
		storage.set(key, value);
	},
	removeItem: (key: string) => {
		storage.delete(key);
	},
};

export const supabase = createClient<Database>(
	env.supabaseUrl,
	env.supabaseAnonKey,
	{
		auth: {
			storage: mmkvStorageAdapter,
			autoRefreshToken: true,
			persistSession: true,
			detectSessionInUrl: false, // Required for React Native
		},
	},
);
```

---

## Auth Context

```tsx
// src/lib/supabase/auth-context.tsx
import { createContext, useContext, useEffect, useState } from "react";
import { AppState } from "react-native";
import type { Session, User } from "@supabase/supabase-js";
import { supabase } from "./client";
import * as SplashScreen from "expo-splash-screen";

interface AuthContextValue {
	session: Session | null;
	user: User | null;
	isLoading: boolean;
	signOut: () => Promise<void>;
}

const AuthContext = createContext<AuthContextValue | null>(null);

export function AuthProvider({ children }: { children: React.ReactNode }) {
	const [session, setSession] = useState<Session | null>(null);
	const [isLoading, setIsLoading] = useState(true);

	useEffect(() => {
		// Get initial session from MMKV (synchronous — no flash)
		supabase.auth.getSession().then(({ data: { session } }) => {
			setSession(session);
			setIsLoading(false);
			SplashScreen.hideAsync();
		});

		const {
			data: { subscription },
		} = supabase.auth.onAuthStateChange((event, session) => {
			setSession(session);
		});

		// Refresh session when app comes back to foreground
		const appStateListener = AppState.addEventListener("change", (state) => {
			if (state === "active") {
				supabase.auth.startAutoRefresh();
			} else {
				supabase.auth.stopAutoRefresh();
			}
		});

		return () => {
			subscription.unsubscribe();
			appStateListener.remove();
		};
	}, []);

	return (
		<AuthContext.Provider
			value={{
				session,
				user: session?.user ?? null,
				isLoading,
				signOut: () => supabase.auth.signOut(),
			}}
		>
			{children}
		</AuthContext.Provider>
	);
}

export function useAuth() {
	const ctx = useContext(AuthContext);
	if (!ctx) throw new Error("useAuth must be used within AuthProvider");
	return ctx;
}

export function useRequiredUser(): User {
	const { user } = useAuth();
	if (!user) throw new Error("Not authenticated");
	return user;
}
```

---

## Query Functions — Never Inline

```ts
// src/lib/queries/tasks.ts
import { supabase } from "@/lib/supabase/client";
import type { Tables, InsertTables, UpdateTables } from "@/lib/supabase/types";

export type Task = Tables<"tasks">;

export async function getTasksByProject(projectId: string): Promise<Task[]> {
	const { data, error } = await supabase
		.from("tasks")
		.select("*")
		.eq("project_id", projectId)
		.order("created_at", { ascending: false });

	if (error) throw new Error(error.message);
	return data;
}

// Use .select() to fetch only needed columns — reduces payload size on mobile
export async function getTaskSummaries(projectId: string) {
	const { data, error } = await supabase
		.from("tasks")
		.select("id, title, is_complete, priority, due_date") // Only what we show in list
		.eq("project_id", projectId)
		.order("created_at", { ascending: false });

	if (error) throw new Error(error.message);
	return data;
}

// Paginated query — always paginate mobile lists
export async function getTasksPaginated(
	projectId: string,
	page: number,
	pageSize = 20,
): Promise<{ tasks: Task[]; hasMore: boolean }> {
	const from = page * pageSize;
	const to = from + pageSize;

	const { data, error, count } = await supabase
		.from("tasks")
		.select("*", { count: "exact" })
		.eq("project_id", projectId)
		.order("created_at", { ascending: false })
		.range(from, to - 1);

	if (error) throw new Error(error.message);
	return {
		tasks: data,
		hasMore: (count ?? 0) > to,
	};
}
```

---

## RLS Policies (Mobile Multi-Tenant)

```sql
-- Standard mobile RLS pattern
-- Users own their data, org members share org data

ALTER TABLE tasks ENABLE ROW LEVEL SECURITY;

-- Users can only see tasks in their organizations
CREATE POLICY "org members can view tasks"
  ON tasks FOR SELECT
  USING (
    project_id IN (
      SELECT p.id FROM projects p
      JOIN org_members om ON om.org_id = p.org_id
      WHERE om.user_id = auth.uid()
    )
  );

-- Users can create tasks in projects they belong to
CREATE POLICY "org members can create tasks"
  ON tasks FOR INSERT
  WITH CHECK (
    created_by = auth.uid()
    AND project_id IN (
      SELECT p.id FROM projects p
      JOIN org_members om ON om.org_id = p.org_id
      WHERE om.user_id = auth.uid()
      AND om.role IN ('owner', 'admin', 'member')
    )
  );

-- Task creator or org admin can update
CREATE POLICY "task owners and admins can update"
  ON tasks FOR UPDATE
  USING (
    created_by = auth.uid()
    OR project_id IN (
      SELECT p.id FROM projects p
      JOIN org_members om ON om.org_id = p.org_id
      WHERE om.user_id = auth.uid() AND om.role IN ('owner', 'admin')
    )
  );

-- Indexes for RLS performance (critical — missing these causes full table scans)
CREATE INDEX tasks_project_id_idx ON tasks(project_id);
CREATE INDEX tasks_created_by_idx ON tasks(created_by);
CREATE INDEX org_members_user_id_idx ON org_members(user_id);
CREATE INDEX org_members_org_id_idx ON org_members(org_id);
```

---

## Realtime Subscriptions (Mobile-Optimized)

```tsx
// src/hooks/use-realtime-tasks.ts
// Only subscribe when screen is in focus — critical for mobile battery life

import { useEffect } from "react";
import { AppState } from "react-native";
import { useQueryClient } from "@tanstack/react-query";
import { useFocusEffect } from "expo-router";
import { useCallback, useRef } from "react";
import { supabase } from "@/lib/supabase/client";
import { queryKeys } from "@/lib/queries/keys";

export function useRealtimeTasks(projectId: string) {
	const queryClient = useQueryClient();
	const channelRef = useRef<ReturnType<typeof supabase.channel> | null>(null);

	useFocusEffect(
		useCallback(() => {
			// Subscribe when screen gains focus
			channelRef.current = supabase
				.channel(`tasks:${projectId}`)
				.on(
					"postgres_changes",
					{
						event: "*",
						schema: "public",
						table: "tasks",
						filter: `project_id=eq.${projectId}`,
					},
					(payload) => {
						// Targeted cache update — avoid full refetch when possible
						if (payload.eventType === "INSERT") {
							queryClient.setQueryData(
								queryKeys.tasks.byProject(projectId),
								(old: Task[] = []) => [payload.new as Task, ...old],
							);
						} else if (payload.eventType === "UPDATE") {
							queryClient.setQueryData(
								queryKeys.tasks.byProject(projectId),
								(old: Task[] = []) =>
									old.map((t) =>
										t.id === payload.new.id ? (payload.new as Task) : t,
									),
							);
						} else if (payload.eventType === "DELETE") {
							queryClient.setQueryData(
								queryKeys.tasks.byProject(projectId),
								(old: Task[] = []) =>
									old.filter((t) => t.id !== payload.old.id),
							);
						}
					},
				)
				.subscribe();

			// Unsubscribe when screen loses focus
			return () => {
				if (channelRef.current) {
					supabase.removeChannel(channelRef.current);
					channelRef.current = null;
				}
			};
		}, [projectId, queryClient]),
	);
}
```

---

## Storage: File Uploads

```ts
// src/lib/mutations/storage.ts
import { supabase } from "@/lib/supabase/client";
import * as ImagePicker from "expo-image-picker";
import * as FileSystem from "expo-file-system";
import { decode } from "base64-arraybuffer";
import { nanoid } from "nanoid/non-secure";

const AVATAR_BUCKET = "avatars";
const MAX_SIZE_BYTES = 2 * 1024 * 1024; // 2MB

export async function pickAndUploadAvatar(userId: string): Promise<string> {
	// Request permission
	const { status } = await ImagePicker.requestMediaLibraryPermissionsAsync();
	if (status !== "granted")
		throw new Error("Media library permission required");

	// Pick image
	const result = await ImagePicker.launchImageLibraryAsync({
		mediaTypes: ImagePicker.MediaTypeOptions.Images,
		allowsEditing: true,
		aspect: [1, 1],
		quality: 0.8, // Compress before upload
		base64: true,
	});

	if (result.canceled || !result.assets[0])
		throw new Error("No image selected");

	const asset = result.assets[0];

	// Validate size
	const fileInfo = await FileSystem.getInfoAsync(asset.uri);
	if (fileInfo.exists && fileInfo.size > MAX_SIZE_BYTES) {
		throw new Error("Image too large. Maximum size is 2MB.");
	}

	const ext = asset.uri.split(".").pop() ?? "jpg";
	const path = `${userId}/${nanoid()}.${ext}`;

	// Upload as base64
	const { error } = await supabase.storage
		.from(AVATAR_BUCKET)
		.upload(path, decode(asset.base64!), {
			contentType: asset.mimeType ?? "image/jpeg",
			upsert: true,
		});

	if (error) throw new Error(error.message);

	const {
		data: { publicUrl },
	} = supabase.storage.from(AVATAR_BUCKET).getPublicUrl(path);

	return publicUrl;
}
```
