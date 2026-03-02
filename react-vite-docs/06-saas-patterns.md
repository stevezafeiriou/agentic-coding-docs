# SaaS & Multi-Tenant Patterns

---

## Database Schema for Multi-Tenancy

```sql
-- Core multi-tenant schema
-- Every user-facing table has org_id FK

CREATE TABLE organizations (
  id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name       text NOT NULL,
  slug       text UNIQUE NOT NULL,
  plan       text NOT NULL DEFAULT 'free', -- free | pro | enterprise
  created_at timestamptz DEFAULT now()
);

CREATE TABLE org_members (
  org_id  uuid REFERENCES organizations(id) ON DELETE CASCADE,
  user_id uuid REFERENCES auth.users(id)    ON DELETE CASCADE,
  role    text NOT NULL DEFAULT 'member',   -- owner | admin | member | viewer
  PRIMARY KEY (org_id, user_id),
  created_at timestamptz DEFAULT now()
);

-- Every feature table follows this pattern:
CREATE TABLE projects (
  id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id     uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  created_by uuid NOT NULL REFERENCES auth.users(id),
  name       text NOT NULL,
  created_at timestamptz DEFAULT now()
);

-- Index for tenant isolation performance
CREATE INDEX projects_org_id_idx ON projects(org_id);
```

---

## Org Context & Switcher

```tsx
// src/hooks/use-user-orgs.ts
export function useUserOrgs() {
	return useQuery({
		queryKey: queryKeys.orgs.byUser(),
		queryFn: async () => {
			const { data, error } = await supabase
				.from("org_members")
				.select("role, organization:organizations(id, name, slug)")
				.order("created_at", { ascending: true });

			if (error) throw new Error(error.message);
			return data;
		},
	});
}
```

```tsx
// src/components/layout/org-switcher.tsx
export function OrgSwitcher() {
	const { data: memberships, isLoading } = useUserOrgs();
	const { activeOrgId, setActiveOrg } = useOrgStore();
	const queryClient = useQueryClient();

	function switchOrg(orgId: string) {
		setActiveOrg(orgId);
		// Clear all org-scoped queries when switching
		queryClient.invalidateQueries();
	}

	const activeOrg = memberships?.find((m) => m.organization.id === activeOrgId);

	return (
		<DropdownMenu>
			<DropdownMenuTrigger asChild>
				<Button variant="ghost" className="w-full justify-between">
					<span className="font-medium">
						{activeOrg?.organization.name ?? "Select org"}
					</span>
					<ChevronsUpDown className="h-4 w-4 text-muted-foreground" />
				</Button>
			</DropdownMenuTrigger>
			<DropdownMenuContent align="start" className="w-56">
				{memberships?.map(({ organization, role }) => (
					<DropdownMenuItem
						key={organization.id}
						onSelect={() => switchOrg(organization.id)}
					>
						<div className="flex items-center gap-2">
							<OrgAvatar name={organization.name} className="h-5 w-5" />
							<span>{organization.name}</span>
							{organization.id === activeOrgId && (
								<Check className="ml-auto h-4 w-4" />
							)}
						</div>
					</DropdownMenuItem>
				))}
				<DropdownMenuSeparator />
				<DropdownMenuItem onSelect={() => navigate("/orgs/new")}>
					<Plus className="mr-2 h-4 w-4" />
					Create organization
				</DropdownMenuItem>
			</DropdownMenuContent>
		</DropdownMenu>
	);
}
```

---

## Role-Based Access Control (RBAC)

```ts
// src/lib/rbac.ts
type OrgRole = "owner" | "admin" | "member" | "viewer";
type Action =
	| "read"
	| "create"
	| "update"
	| "delete"
	| "manage_members"
	| "manage_billing";

const PERMISSIONS: Record<OrgRole, Action[]> = {
	owner: [
		"read",
		"create",
		"update",
		"delete",
		"manage_members",
		"manage_billing",
	],
	admin: ["read", "create", "update", "delete", "manage_members"],
	member: ["read", "create", "update"],
	viewer: ["read"],
};

export function can(role: OrgRole | null | undefined, action: Action): boolean {
	if (!role) return false;
	return PERMISSIONS[role].includes(action);
}
```

```tsx
// src/hooks/use-org-role.ts
export function useOrgRole(orgId: string): OrgRole | null {
	const { data } = useUserOrgs();
	return data?.find((m) => m.organization.id === orgId)?.role ?? null;
}

// Usage in components
export function DeleteProjectButton({ projectId, orgId }: Props) {
	const role = useOrgRole(orgId);

	if (!can(role, "delete")) return null;

	return (
		<Button variant="destructive" onClick={() => deleteProject(projectId)}>
			Delete
		</Button>
	);
}
```

---

## Subscription / Plan Gating

```tsx
// src/components/billing/plan-gate.tsx
interface PlanGateProps {
	requiredPlan: "pro" | "enterprise";
	children: React.ReactNode;
	fallback?: React.ReactNode;
}

export function PlanGate({ requiredPlan, children, fallback }: PlanGateProps) {
	const orgId = useActiveOrgId();
	const { data: org } = useOrgDetail(orgId);

	const planRank = { free: 0, pro: 1, enterprise: 2 };
	const hasAccess =
		(planRank[org?.plan ?? "free"] ?? 0) >= planRank[requiredPlan];

	if (!hasAccess) {
		return fallback ? (
			<>{fallback}</>
		) : (
			<UpgradeBanner requiredPlan={requiredPlan} />
		);
	}

	return <>{children}</>;
}

// Usage
<PlanGate
	requiredPlan="pro"
	fallback={<UpgradeCTA feature="Advanced analytics" />}
>
	<AnalyticsDashboard />
</PlanGate>;
```

---

## Invitation Flow

```ts
// Database table for pending invites
CREATE TABLE org_invites (
  id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id     uuid NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
  email      text NOT NULL,
  role       text NOT NULL DEFAULT 'member',
  token      text UNIQUE NOT NULL DEFAULT encode(gen_random_bytes(32), 'hex'),
  invited_by uuid REFERENCES auth.users(id),
  expires_at timestamptz DEFAULT now() + interval '7 days',
  accepted_at timestamptz,
  created_at timestamptz DEFAULT now()
);
```

```tsx
// Accept invite page
export function AcceptInvitePage() {
	const [token] = useSearchParams();
	const navigate = useNavigate();
	const { user } = useAuth();

	const acceptMutation = useMutation({
		mutationFn: async () => {
			const { data, error } = await supabase
				.from("org_invites")
				.select("*, organization:organizations(id, name)")
				.eq("token", token.get("token"))
				.gt("expires_at", new Date().toISOString())
				.is("accepted_at", null)
				.single();

			if (error || !data) throw new Error("Invalid or expired invite");

			// Add to org
			await supabase.from("org_members").insert({
				org_id: data.org_id,
				user_id: user!.id,
				role: data.role,
			});

			// Mark as accepted
			await supabase
				.from("org_invites")
				.update({ accepted_at: new Date().toISOString() })
				.eq("id", data.id);

			return data.organization;
		},
		onSuccess: (org) => navigate(`/dashboard/${org.id}`),
	});
	// ...
}
```

---

## Audit Log Pattern

```sql
CREATE TABLE audit_logs (
  id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id     uuid NOT NULL REFERENCES organizations(id),
  user_id    uuid REFERENCES auth.users(id),
  action     text NOT NULL,         -- 'project.created', 'member.removed', etc.
  entity_type text NOT NULL,        -- 'project', 'member', 'invoice'
  entity_id  uuid,
  metadata   jsonb DEFAULT '{}',
  created_at timestamptz DEFAULT now()
);

-- RLS: org admins can read, nobody can write directly (use trigger/function)
ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;

CREATE POLICY "org admins can read audit logs"
  ON audit_logs FOR SELECT
  USING (
    org_id IN (
      SELECT org_id FROM org_members
      WHERE user_id = auth.uid() AND role IN ('owner', 'admin')
    )
  );
```

```ts
// src/lib/audit.ts
export async function logAuditEvent(params: {
	orgId: string;
	action: string;
	entityType: string;
	entityId?: string;
	metadata?: Record<string, unknown>;
}) {
	// Fire-and-forget — don't await in critical paths
	supabase
		.from("audit_logs")
		.insert({
			org_id: params.orgId,
			action: params.action,
			entity_type: params.entityType,
			entity_id: params.entityId,
			metadata: params.metadata ?? {},
		})
		.then(({ error }) => {
			if (error) console.error("Audit log failed:", error);
		});
}
```
