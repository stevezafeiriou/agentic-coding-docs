# Vite Build Tooling & Deployment

---

## Vite Config (Production-Ready)

```ts
// vite.config.ts
import { defineConfig, loadEnv } from "vite";
import react from "@vitejs/plugin-react";
import tsconfigPaths from "vite-tsconfig-paths";
import { visualizer } from "rollup-plugin-visualizer";

export default defineConfig(({ mode }) => {
	const env = loadEnv(mode, process.cwd(), "");

	return {
		plugins: [
			react(),
			tsconfigPaths(), // Enables @/ path aliases from tsconfig
			mode === "analyze" &&
				visualizer({
					open: true,
					gzipSize: true,
					brotliSize: true,
				}),
		].filter(Boolean),

		build: {
			target: "es2020",
			sourcemap: mode !== "production", // No sourcemaps in prod
			rollupOptions: {
				output: {
					// Manual chunk splitting for optimal caching
					manualChunks: {
						// Vendor: rarely changes → long cache lifetime
						"vendor-react": ["react", "react-dom", "react-router-dom"],
						"vendor-query": ["@tanstack/react-query"],
						"vendor-ui": [
							"@radix-ui/react-dialog",
							"@radix-ui/react-dropdown-menu",
						],
						"vendor-forms": ["react-hook-form", "@hookform/resolvers", "zod"],
						"vendor-supabase": ["@supabase/supabase-js"],
					},
					// Content-hash filenames for immutable caching
					assetFileNames: "assets/[name]-[hash][extname]",
					chunkFileNames: "chunks/[name]-[hash].js",
					entryFileNames: "[name]-[hash].js",
				},
			},
		},

		optimizeDeps: {
			include: [
				"react",
				"react-dom",
				"@tanstack/react-query",
				"@supabase/supabase-js",
			],
		},

		server: {
			port: 3000,
			strictPort: true,
		},
	};
});
```

---

## TypeScript Config (Strict)

```json
// tsconfig.json
{
	"compilerOptions": {
		"target": "ES2020",
		"lib": ["ES2020", "DOM", "DOM.Iterable"],
		"module": "ESNext",
		"moduleResolution": "bundler",
		"jsx": "react-jsx",

		// Strictness — all on
		"strict": true,
		"noUnusedLocals": true,
		"noUnusedParameters": true,
		"noFallthroughCasesInSwitch": true,
		"noUncheckedIndexedAccess": true,

		// Path aliases
		"baseUrl": ".",
		"paths": { "@/*": ["./src/*"] },

		"skipLibCheck": true,
		"resolveJsonModule": true,
		"isolatedModules": true,
		"noEmit": true
	},
	"include": ["src"],
	"references": [{ "path": "./tsconfig.node.json" }]
}
```

---

## Environment Variables

```bash
# .env.local (never commit)
VITE_SUPABASE_URL=https://xxxx.supabase.co
VITE_SUPABASE_ANON_KEY=eyJh...

# .env.example (commit — documents required vars)
VITE_SUPABASE_URL=
VITE_SUPABASE_ANON_KEY=
```

```ts
// src/lib/env.ts — Validate env at startup, not scattered through the codebase
import { z } from "zod";

const envSchema = z.object({
	VITE_SUPABASE_URL: z.string().url(),
	VITE_SUPABASE_ANON_KEY: z.string().min(1),
});

// Throws at startup if env is misconfigured — fails fast
export const env = envSchema.parse(import.meta.env);
```

---

## Project Scripts

```json
// package.json
{
	"scripts": {
		"dev": "vite",
		"build": "tsc -b && vite build",
		"preview": "vite preview",
		"typecheck": "tsc --noEmit",
		"lint": "eslint src --ext .ts,.tsx --max-warnings 0",
		"analyze": "VITE_MODE=analyze vite build",
		"db:types": "npx supabase gen types typescript --local > src/lib/supabase/generated.ts",
		"db:migrate": "npx supabase db push",
		"db:reset": "npx supabase db reset"
	}
}
```

---

## Deployment (Vercel — Recommended)

```json
// vercel.json
{
	"rewrites": [{ "source": "/(.*)", "destination": "/index.html" }],
	"headers": [
		{
			"source": "/assets/(.*)",
			"headers": [
				{
					"key": "Cache-Control",
					"value": "public, max-age=31536000, immutable"
				}
			]
		},
		{
			"source": "/(.*)",
			"headers": [
				{ "key": "X-Content-Type-Options", "value": "nosniff" },
				{ "key": "X-Frame-Options", "value": "DENY" },
				{ "key": "X-XSS-Protection", "value": "1; mode=block" }
			]
		}
	]
}
```

### CI/CD with GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push: { branches: [main] }
  pull_request: { branches: [main] }

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: "20", cache: "npm" }
      - run: npm ci
      - run: npm run typecheck
      - run: npm run lint
      - run: npm run build
```
