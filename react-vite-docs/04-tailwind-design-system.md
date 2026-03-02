# Tailwind CSS + Design System

---

## Config: Design Tokens First

Everything in the design system flows from `tailwind.config.ts`.
Never hardcode colors, spacing, or typography inline — use tokens.

```ts
// tailwind.config.ts
import type { Config } from "tailwindcss";

export default {
	darkMode: ["class"],
	content: ["./index.html", "./src/**/*.{ts,tsx}"],
	theme: {
		extend: {
			colors: {
				// shadcn/ui CSS variable tokens — support dark mode automatically
				border: "hsl(var(--border))",
				input: "hsl(var(--input))",
				ring: "hsl(var(--ring))",
				background: "hsl(var(--background))",
				foreground: "hsl(var(--foreground))",

				primary: {
					DEFAULT: "hsl(var(--primary))",
					foreground: "hsl(var(--primary-foreground))",
				},
				secondary: {
					DEFAULT: "hsl(var(--secondary))",
					foreground: "hsl(var(--secondary-foreground))",
				},
				destructive: {
					DEFAULT: "hsl(var(--destructive))",
					foreground: "hsl(var(--destructive-foreground))",
				},
				muted: {
					DEFAULT: "hsl(var(--muted))",
					foreground: "hsl(var(--muted-foreground))",
				},
				accent: {
					DEFAULT: "hsl(var(--accent))",
					foreground: "hsl(var(--accent-foreground))",
				},
				card: {
					DEFAULT: "hsl(var(--card))",
					foreground: "hsl(var(--card-foreground))",
				},
			},
			borderRadius: {
				lg: "var(--radius)",
				md: "calc(var(--radius) - 2px)",
				sm: "calc(var(--radius) - 4px)",
			},
		},
	},
	plugins: [require("tailwindcss-animate")],
} satisfies Config;
```

### CSS Variables (in `index.css`)

```css
@layer base {
	:root {
		--background: 0 0% 100%;
		--foreground: 222.2 84% 4.9%;
		--card: 0 0% 100%;
		--card-foreground: 222.2 84% 4.9%;
		--primary: 221.2 83.2% 53.3%;
		--primary-foreground: 210 40% 98%;
		--muted: 210 40% 96.1%;
		--muted-foreground: 215.4 16.3% 46.9%;
		--destructive: 0 84.2% 60.2%;
		--border: 214.3 31.8% 91.4%;
		--radius: 0.5rem;
	}

	.dark {
		--background: 222.2 84% 4.9%;
		--foreground: 210 40% 98%;
		/* ... dark variants */
	}
}
```

---

## The `cn()` Utility — Always Use It

```ts
// src/utils/cn.ts
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

// Use this for ALL className merging — it handles Tailwind conflicts correctly
export function cn(...inputs: ClassValue[]) {
	return twMerge(clsx(inputs));
}
```

---

## Component Variants with `cva`

Use `class-variance-authority` for components with multiple visual variants.
Never string-concatenate conditional class logic.

```tsx
// src/components/ui/badge.tsx
import { cva, type VariantProps } from "class-variance-authority";
import { cn } from "@/utils/cn";

const badgeVariants = cva(
	// Base classes — always applied
	"inline-flex items-center rounded-full px-2.5 py-0.5 text-xs font-medium transition-colors",
	{
		variants: {
			variant: {
				default: "bg-primary text-primary-foreground",
				secondary: "bg-secondary text-secondary-foreground",
				destructive: "bg-destructive text-destructive-foreground",
				outline: "border border-input bg-background text-foreground",
				success:
					"bg-green-100 text-green-800 dark:bg-green-900 dark:text-green-200",
				warning:
					"bg-yellow-100 text-yellow-800 dark:bg-yellow-900 dark:text-yellow-200",
			},
		},
		defaultVariants: {
			variant: "default",
		},
	},
);

interface BadgeProps
	extends React.HTMLAttributes<HTMLDivElement>,
		VariantProps<typeof badgeVariants> {}

export function Badge({ className, variant, ...props }: BadgeProps) {
	return (
		<div className={cn(badgeVariants({ variant }), className)} {...props} />
	);
}
```

---

## Layout Patterns for SaaS / Dashboards

### App Shell

```tsx
// src/components/layout/app-shell.tsx
export function AppShell({ children }: { children: React.ReactNode }) {
	const { sidebarOpen } = useUIStore();

	return (
		<div className="flex h-screen overflow-hidden bg-background">
			{/* Sidebar */}
			<aside
				className={cn(
					"flex flex-col border-r bg-card transition-all duration-200",
					sidebarOpen ? "w-60" : "w-16",
				)}
			>
				<Sidebar />
			</aside>

			{/* Main content */}
			<div className="flex flex-1 flex-col overflow-hidden">
				<header className="flex h-14 items-center border-b px-6">
					<TopBar />
				</header>
				<main className="flex-1 overflow-y-auto p-6">{children}</main>
			</div>
		</div>
	);
}
```

### Page Header Pattern

```tsx
// src/components/layout/page-header.tsx
interface PageHeaderProps {
	title: string;
	description?: string;
	action?: React.ReactNode;
}

export function PageHeader({ title, description, action }: PageHeaderProps) {
	return (
		<div className="flex items-start justify-between">
			<div className="space-y-1">
				<h1 className="text-2xl font-semibold tracking-tight text-foreground">
					{title}
				</h1>
				{description && (
					<p className="text-sm text-muted-foreground">{description}</p>
				)}
			</div>
			{action && <div className="flex items-center gap-2">{action}</div>}
		</div>
	);
}
```

---

## Consistent Empty, Loading & Error States

Define these once — never ad-hoc per component.

```tsx
// src/components/ui/empty-state.tsx
interface EmptyStateProps {
	icon: React.ElementType;
	title: string;
	description: string;
	action?: React.ReactNode;
}

export function EmptyState({
	icon: Icon,
	title,
	description,
	action,
}: EmptyStateProps) {
	return (
		<div className="flex flex-col items-center justify-center rounded-lg border border-dashed p-8 text-center">
			<div className="flex h-12 w-12 items-center justify-center rounded-full bg-muted">
				<Icon className="h-6 w-6 text-muted-foreground" />
			</div>
			<h3 className="mt-4 text-sm font-medium text-foreground">{title}</h3>
			<p className="mt-1 text-sm text-muted-foreground">{description}</p>
			{action && <div className="mt-4">{action}</div>}
		</div>
	);
}

// src/components/ui/skeleton.tsx — use for loading states
// Always skeleton-match the real content layout
export function TableSkeleton({ rows = 5 }: { rows?: number }) {
	return (
		<div className="space-y-2">
			{Array.from({ length: rows }).map((_, i) => (
				<div key={i} className="flex gap-3 rounded-md border p-4">
					<div className="h-8 w-8 animate-pulse rounded-full bg-muted" />
					<div className="flex-1 space-y-2">
						<div className="h-4 w-1/3 animate-pulse rounded bg-muted" />
						<div className="h-3 w-1/2 animate-pulse rounded bg-muted" />
					</div>
				</div>
			))}
		</div>
	);
}
```

---

## Typography Scale

Use these classes consistently — never mix font-size utilities ad-hoc.

```
Page title:        text-2xl font-semibold tracking-tight
Section heading:   text-lg font-medium
Card title:        text-base font-medium
Body text:         text-sm text-foreground
Secondary text:    text-sm text-muted-foreground
Captions/labels:   text-xs text-muted-foreground
Code:              font-mono text-sm
```

---

## Spacing & Layout Rules

- Use `space-y-*` for vertical stacks inside a component
- Use `gap-*` for flex/grid containers
- Page padding: `p-6` or `px-6 py-4`
- Card padding: `p-4` or `p-6` for larger cards
- Section gaps: `space-y-6` between major sections, `space-y-4` within a section
- Never use arbitrary values `[123px]` — if needed, add to config
