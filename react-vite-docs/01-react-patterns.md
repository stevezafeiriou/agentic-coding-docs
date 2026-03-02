# React + TypeScript Patterns

---

## Component Architecture

### Three-Tier Component Classification

Every component belongs to one of three tiers. Misclassifying causes
premature abstraction or unnecessary duplication.

**Tier 1: UI Primitives** (`src/components/ui/`)

- No business logic. No data fetching. No Supabase.
- Accepts all data via props. Fully controlled.
- Examples: Button, Input, Card, Badge, Modal, Tooltip
- Source: shadcn/ui (copy into repo — never import from node_modules)

**Tier 2: Feature Components** (`src/components/[feature]/`)

- Contains business logic for one domain.
- May own local state. May call custom hooks.
- Never fetches data directly — receives it from a container or hook.
- Examples: UserAvatar, OrgSwitcher, InvoiceRow

**Tier 3: Page Components** (`src/app/[route]/page.tsx`)

- Orchestrates feature components.
- Owns data fetching (via TanStack Query hooks).
- Renders loading/error/empty states.
- Handles navigation.

---

## Component Template

```tsx
// src/components/features/user-card.tsx
import type { Database } from "@/lib/supabase/types";

// 1. Derive types from database types — never duplicate
type User = Database["public"]["Tables"]["users"]["Row"];

interface UserCardProps {
	user: User;
	onEdit?: (id: string) => void;
	className?: string;
}

// 2. Named export always (default export only for pages)
export function UserCard({ user, onEdit, className }: UserCardProps) {
	return (
		<div className={cn("rounded-lg border bg-card p-4", className)}>
			<div className="flex items-center gap-3">
				<Avatar src={user.avatar_url} fallback={user.full_name[0]} />
				<div>
					<p className="font-medium text-foreground">{user.full_name}</p>
					<p className="text-sm text-muted-foreground">{user.email}</p>
				</div>
			</div>
			{onEdit && (
				<Button
					variant="ghost"
					size="sm"
					onClick={() => onEdit(user.id)}
					aria-label={`Edit ${user.full_name}`}
				>
					<Pencil className="h-4 w-4" />
				</Button>
			)}
		</div>
	);
}
```

---

## Custom Hook Patterns

### Data Fetching Hook (TanStack Query + Supabase)

```tsx
// src/hooks/use-organization-members.ts
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { getOrgMembers, removeOrgMember } from "@/lib/queries/organizations";
import { toast } from "sonner";

// Query keys: always arrays, always co-located with the hook
const orgMemberKeys = {
	all: ["org-members"] as const,
	byOrg: (orgId: string) => [...orgMemberKeys.all, orgId] as const,
};

export function useOrgMembers(orgId: string) {
	return useQuery({
		queryKey: orgMemberKeys.byOrg(orgId),
		queryFn: () => getOrgMembers(orgId),
		enabled: !!orgId,
		staleTime: 1000 * 60 * 5, // 5 minutes
	});
}

export function useRemoveOrgMember(orgId: string) {
	const queryClient = useQueryClient();

	return useMutation({
		mutationFn: (userId: string) => removeOrgMember(orgId, userId),

		// Optimistic update
		onMutate: async (userId) => {
			await queryClient.cancelQueries({ queryKey: orgMemberKeys.byOrg(orgId) });
			const previous = queryClient.getQueryData(orgMemberKeys.byOrg(orgId));

			queryClient.setQueryData(orgMemberKeys.byOrg(orgId), (old: Member[]) =>
				old.filter((m) => m.user_id !== userId),
			);

			return { previous };
		},

		onError: (_, __, context) => {
			// Rollback on error
			queryClient.setQueryData(orgMemberKeys.byOrg(orgId), context?.previous);
			toast.error("Failed to remove member");
		},

		onSuccess: () => {
			toast.success("Member removed");
		},

		onSettled: () => {
			// Always refetch to ensure consistency
			queryClient.invalidateQueries({ queryKey: orgMemberKeys.byOrg(orgId) });
		},
	});
}
```

### Local State Hook

```tsx
// src/hooks/use-disclosure.ts
// Canonical pattern for modals, dropdowns, drawers
import { useState, useCallback } from "react";

export function useDisclosure(defaultOpen = false) {
	const [isOpen, setIsOpen] = useState(defaultOpen);

	return {
		isOpen,
		open: useCallback(() => setIsOpen(true), []),
		close: useCallback(() => setIsOpen(false), []),
		toggle: useCallback(() => setIsOpen((p) => !p), []),
		onOpenChange: setIsOpen,
	};
}
```

### Debounced Search Hook

```tsx
// src/hooks/use-debounced-search.ts
import { useState, useEffect } from "react";
import { useQuery } from "@tanstack/react-query";
import { searchUsers } from "@/lib/queries/users";

export function useDebouncedSearch(delay = 300) {
	const [raw, setRaw] = useState("");
	const [debounced, setDebounced] = useState("");

	useEffect(() => {
		const timer = setTimeout(() => setDebounced(raw), delay);
		return () => clearTimeout(timer);
	}, [raw, delay]);

	const query = useQuery({
		queryKey: ["search", "users", debounced],
		queryFn: () => searchUsers(debounced),
		enabled: debounced.length >= 2,
		staleTime: 1000 * 30,
	});

	return { query: raw, setQuery: setRaw, ...query };
}
```

---

## Form Pattern (React Hook Form + Zod)

```tsx
// src/lib/validations/invite-member.ts
import { z } from "zod";

export const inviteMemberSchema = z.object({
	email: z.string().email("Please enter a valid email"),
	role: z.enum(["admin", "member", "viewer"], {
		errorMap: () => ({ message: "Please select a role" }),
	}),
});

export type InviteMemberInput = z.infer<typeof inviteMemberSchema>;
```

```tsx
// src/components/features/invite-member-form.tsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import {
	inviteMemberSchema,
	type InviteMemberInput,
} from "@/lib/validations/invite-member";
import { useInviteMember } from "@/hooks/use-invite-member";

interface InviteMemberFormProps {
	orgId: string;
	onSuccess?: () => void;
}

export function InviteMemberForm({ orgId, onSuccess }: InviteMemberFormProps) {
	const { mutate, isPending } = useInviteMember(orgId);

	const form = useForm<InviteMemberInput>({
		resolver: zodResolver(inviteMemberSchema),
		defaultValues: { email: "", role: "member" },
	});

	function onSubmit(data: InviteMemberInput) {
		mutate(data, { onSuccess });
	}

	return (
		<form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
			<div className="space-y-1.5">
				<Label htmlFor="email">Email address</Label>
				<Input
					id="email"
					type="email"
					placeholder="colleague@company.com"
					aria-invalid={!!form.formState.errors.email}
					{...form.register("email")}
				/>
				{form.formState.errors.email && (
					<p className="text-sm text-destructive" role="alert">
						{form.formState.errors.email.message}
					</p>
				)}
			</div>

			<div className="space-y-1.5">
				<Label htmlFor="role">Role</Label>
				<Select
					onValueChange={(v) =>
						form.setValue("role", v as InviteMemberInput["role"])
					}
				>
					<SelectTrigger id="role" aria-invalid={!!form.formState.errors.role}>
						<SelectValue placeholder="Select a role" />
					</SelectTrigger>
					<SelectContent>
						<SelectItem value="admin">Admin</SelectItem>
						<SelectItem value="member">Member</SelectItem>
						<SelectItem value="viewer">Viewer (read only)</SelectItem>
					</SelectContent>
				</Select>
			</div>

			<Button type="submit" disabled={isPending} className="w-full">
				{isPending ? "Sending invite…" : "Send invite"}
			</Button>
		</form>
	);
}
```

---

## Async State Rendering Pattern

Always render all three states explicitly. Never show a blank screen.

```tsx
export function MembersPage() {
	const { orgId } = useParams<{ orgId: string }>();
	const { data, isLoading, isError, error } = useOrgMembers(orgId!);

	if (isLoading) return <MembersSkeleton />;

	if (isError)
		return (
			<ErrorState
				title="Couldn't load members"
				description={error.message}
				action={{ label: "Try again", onClick: () => window.location.reload() }}
			/>
		);

	if (!data || data.length === 0)
		return (
			<EmptyState
				icon={Users}
				title="No members yet"
				description="Invite your team to get started."
				action={<InviteMemberButton orgId={orgId!} />}
			/>
		);

	return (
		<div className="space-y-4">
			{data.map((member) => (
				<MemberRow key={member.user_id} member={member} />
			))}
		</div>
	);
}
```

---

## Memoization Rules

Only add `useMemo`, `useCallback`, `memo` after profiling shows a problem.
Premature memoization adds complexity without benefit.

**DO memoize:**

- Expensive derivations (sorting/filtering large lists)
- Stable callback references passed to memoized children
- Context values that would re-render all consumers

**DON'T memoize:**

- Simple components that render fast
- Primitive values
- Functions that are only used in the same component

```tsx
// DO: expensive derivation
const sortedMembers = useMemo(
	() => [...members].sort((a, b) => a.full_name.localeCompare(b.full_name)),
	[members],
);

// DON'T: this is fine without memo
const displayName = `${user.first_name} ${user.last_name}`;
```
