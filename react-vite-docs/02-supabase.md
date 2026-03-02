# Supabase Patterns: Auth, RLS, Realtime, Storage

---

## Client Setup

### Browser Client (Singleton)

```ts
// src/lib/supabase/client.ts
import { createBrowserClient } from "@supabase/ssr";
import type { Database } from "./types";

// Singleton — import this everywhere in client components
export const supabase = createBrowserClient<Database>(
	import.meta.env.VITE_SUPABASE_URL,
	import.meta.env.VITE_SUPABASE_ANON_KEY,
);
```

### Always Use Generated Types

```ts
// src/lib/supabase/types.ts
// Auto-generate with: npx supabase gen types typescript --local > src/lib/supabase/types.ts
export type { Database } from "./generated"; // commit the generated file

// Convenience type helpers
export type Tables<T extends keyof Database["public"]["Tables"]> =
	Database["public"]["Tables"][T]["Row"];

export type InsertTables<T extends keyof Database["public"]["Tables"]> =
	Database["public"]["Tables"][T]["Insert"];

export type UpdateTables<T extends keyof Database["public"]["Tables"]> =
	Database["public"]["Tables"][T]["Update"];

export type Enums<T extends keyof Database["public"]["Enums"]> =
	Database["public"]["Enums"][T];
```

---

## Query Functions (Never Write Inline)

All Supabase queries live in `/lib/queries/`. Components call functions, never chains.

```ts
// src/lib/queries/projects.ts
import { supabase } from "@/lib/supabase/client";
import type { Tables, InsertTables, UpdateTables } from "@/lib/supabase/types";

export type Project = Tables<"projects">;

// Always throw on error — TanStack Query handles it
export async function getProjectsByOrg(orgId: string): Promise<Project[]> {
	const { data, error } = await supabase
		.from("projects")
		.select("*")
		.eq("org_id", orgId)
		.order("created_at", { ascending: false });

	if (error) throw new Error(error.message);
	return data;
}

export async function getProjectById(id: string): Promise<Project> {
	const { data, error } = await supabase
		.from("projects")
		.select("*, owner:users(id, full_name, avatar_url)")
		.eq("id", id)
		.single();

	if (error) throw new Error(error.message);
	return data;
}

export async function createProject(
	input: InsertTables<"projects">,
): Promise<Project> {
	const { data, error } = await supabase
		.from("projects")
		.insert(input)
		.select()
		.single();

	if (error) throw new Error(error.message);
	return data;
}

export async function updateProject(
	id: string,
	input: UpdateTables<"projects">,
): Promise<Project> {
	const { data, error } = await supabase
		.from("projects")
		.update(input)
		.eq("id", id)
		.select()
		.single();

	if (error) throw new Error(error.message);
	return data;
}

export async function deleteProject(id: string): Promise<void> {
	const { error } = await supabase.from("projects").delete().eq("id", id);

	if (error) throw new Error(error.message);
}
```

---

## Row Level Security (RLS)

RLS is the **only** security layer. Frontend filtering is cosmetic only.

### Standard Multi-Tenant Pattern

```sql
-- migrations/001_projects_rls.sql

-- Enable RLS (always)
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

-- Helper: get the current user's org memberships
CREATE OR REPLACE FUNCTION get_user_org_ids()
RETURNS uuid[] AS $$
  SELECT array_agg(org_id)
  FROM org_members
  WHERE user_id = auth.uid()
$$ LANGUAGE sql SECURITY DEFINER STABLE;

-- Read: org members can see org projects
CREATE POLICY "org members can read projects"
  ON projects FOR SELECT
  USING (org_id = ANY(get_user_org_ids()));

-- Insert: org admins/members can create projects
CREATE POLICY "org members can create projects"
  ON projects FOR INSERT
  WITH CHECK (
    org_id IN (
      SELECT org_id FROM org_members
      WHERE user_id = auth.uid()
      AND role IN ('admin', 'member')
    )
  );

-- Update: project owner or org admin
CREATE POLICY "owners and admins can update projects"
  ON projects FOR UPDATE
  USING (
    owner_id = auth.uid()
    OR
    org_id IN (
      SELECT org_id FROM org_members
      WHERE user_id = auth.uid() AND role = 'admin'
    )
  )
  WITH CHECK (
    owner_id = auth.uid()
    OR
    org_id IN (
      SELECT org_id FROM org_members
      WHERE user_id = auth.uid() AND role = 'admin'
    )
  );

-- Delete: org admin only
CREATE POLICY "only admins can delete projects"
  ON projects FOR DELETE
  USING (
    org_id IN (
      SELECT org_id FROM org_members
      WHERE user_id = auth.uid() AND role = 'admin'
    )
  );
```

### Profile Auto-Creation Trigger

```sql
-- migrations/002_user_profiles.sql

CREATE TABLE public.profiles (
  id         uuid PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  full_name  text,
  avatar_url text,
  created_at timestamptz DEFAULT now()
);

ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "users can read all profiles"
  ON profiles FOR SELECT USING (true);

CREATE POLICY "users can update own profile"
  ON profiles FOR UPDATE USING (id = auth.uid());

-- Auto-create profile on signup
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS trigger AS $$
BEGIN
  INSERT INTO public.profiles (id, full_name, avatar_url)
  VALUES (
    new.id,
    new.raw_user_meta_data->>'full_name',
    new.raw_user_meta_data->>'avatar_url'
  );
  RETURN new;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();
```

---

## Authentication Patterns

### Auth Context

```tsx
// src/lib/supabase/auth-context.tsx
import { createContext, useContext, useEffect, useState } from "react";
import type { Session, User } from "@supabase/supabase-js";
import { supabase } from "./client";

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
		// Get initial session
		supabase.auth.getSession().then(({ data: { session } }) => {
			setSession(session);
			setIsLoading(false);
		});

		// Listen for changes
		const {
			data: { subscription },
		} = supabase.auth.onAuthStateChange((_, session) => {
			setSession(session);
		});

		return () => subscription.unsubscribe();
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

// Convenience — throws if not authenticated
export function useRequiredUser(): User {
	const { user } = useAuth();
	if (!user) throw new Error("User is required");
	return user;
}
```

### Protected Route

```tsx
// src/components/auth/protected-route.tsx
import { Navigate, useLocation } from "react-router-dom";
import { useAuth } from "@/lib/supabase/auth-context";

export function ProtectedRoute({ children }: { children: React.ReactNode }) {
	const { session, isLoading } = useAuth();
	const location = useLocation();

	if (isLoading) return <FullPageSpinner />;

	if (!session) {
		return <Navigate to="/login" state={{ from: location }} replace />;
	}

	return <>{children}</>;
}
```

---

## Realtime Subscriptions

```tsx
// src/hooks/use-realtime-project.ts
import { useEffect } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { supabase } from "@/lib/supabase/client";

export function useRealtimeProject(projectId: string) {
	const queryClient = useQueryClient();

	useEffect(() => {
		const channel = supabase
			.channel(`project:${projectId}`)
			.on(
				"postgres_changes",
				{
					event: "*", // INSERT | UPDATE | DELETE
					schema: "public",
					table: "tasks",
					filter: `project_id=eq.${projectId}`,
				},
				() => {
					// Invalidate and refetch — don't try to merge manually
					queryClient.invalidateQueries({ queryKey: ["tasks", projectId] });
				},
			)
			.subscribe();

		return () => {
			supabase.removeChannel(channel);
		};
	}, [projectId, queryClient]);
}
```

---

## Storage Patterns

```ts
// src/lib/queries/storage.ts
import { supabase } from "@/lib/supabase/client";
import { nanoid } from "nanoid";

const BUCKET = "avatars";
const MAX_FILE_SIZE = 2 * 1024 * 1024; // 2MB

export async function uploadAvatar(
	userId: string,
	file: File,
): Promise<string> {
	if (file.size > MAX_FILE_SIZE) {
		throw new Error("File too large. Maximum size is 2MB.");
	}

	if (!file.type.startsWith("image/")) {
		throw new Error("File must be an image.");
	}

	const ext = file.name.split(".").pop();
	const path = `${userId}/${nanoid()}.${ext}`;

	const { error } = await supabase.storage
		.from(BUCKET)
		.upload(path, file, { upsert: true });

	if (error) throw new Error(error.message);

	const {
		data: { publicUrl },
	} = supabase.storage.from(BUCKET).getPublicUrl(path);

	return publicUrl;
}

export async function deleteAvatar(path: string): Promise<void> {
	const { error } = await supabase.storage.from(BUCKET).remove([path]);
	if (error) throw new Error(error.message);
}
```

---

## Edge Functions Pattern

```ts
// supabase/functions/send-invite/index.ts
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";
import { corsHeaders } from "../_shared/cors.ts";

Deno.serve(async (req) => {
	if (req.method === "OPTIONS")
		return new Response("ok", { headers: corsHeaders });

	try {
		const { email, orgId } = await req.json();

		// Use service role key for admin operations
		const supabase = createClient(
			Deno.env.get("SUPABASE_URL")!,
			Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!,
		);

		// Verify caller is an org admin
		const authHeader = req.headers.get("Authorization")!;
		const {
			data: { user },
		} = await supabase.auth.getUser(authHeader.replace("Bearer ", ""));

		const { data: membership } = await supabase
			.from("org_members")
			.select("role")
			.eq("org_id", orgId)
			.eq("user_id", user!.id)
			.single();

		if (membership?.role !== "admin") {
			return new Response("Forbidden", { status: 403, headers: corsHeaders });
		}

		// ... send invite logic

		return new Response(JSON.stringify({ ok: true }), {
			headers: { ...corsHeaders, "Content-Type": "application/json" },
		});
	} catch (err) {
		return new Response(JSON.stringify({ error: err.message }), {
			status: 500,
			headers: { ...corsHeaders, "Content-Type": "application/json" },
		});
	}
});
```
