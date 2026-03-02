# Expo Architecture: Project Setup, Navigation & Config

---

## app.config.ts (Dynamic Config)

Use `app.config.ts` (not `app.json`) for all production apps — it supports
dynamic values and environment variables.

```ts
// app.config.ts
import { ExpoConfig, ConfigContext } from "expo/config";

export default ({ config }: ConfigContext): ExpoConfig => ({
	...config,
	name: process.env.APP_ENV === "production" ? "MyApp" : "MyApp (Dev)",
	slug: "myapp",
	version: "1.0.0",
	orientation: "portrait",
	icon: "./assets/icon.png",
	userInterfaceStyle: "automatic", // Supports dark mode
	splash: {
		image: "./assets/splash.png",
		resizeMode: "contain",
		backgroundColor: "#ffffff",
	},
	ios: {
		supportsTablet: true,
		bundleIdentifier: "com.yourcompany.myapp",
		infoPlist: {
			UIBackgroundModes: ["remote-notification"],
		},
	},
	android: {
		adaptiveIcon: {
			foregroundImage: "./assets/adaptive-icon.png",
			backgroundColor: "#ffffff",
		},
		package: "com.yourcompany.myapp",
		googleServicesFile: process.env.GOOGLE_SERVICES_JSON,
	},
	plugins: [
		"expo-router",
		"expo-secure-store",
		[
			"expo-notifications",
			{
				icon: "./assets/notification-icon.png",
				color: "#ffffff",
				sounds: [],
			},
		],
		[
			"expo-build-properties",
			{
				ios: { deploymentTarget: "15.1" },
				android: { compileSdkVersion: 34, targetSdkVersion: 34 },
			},
		],
	],
	experiments: {
		typedRoutes: true, // Typed Expo Router paths
	},
	extra: {
		supabaseUrl: process.env.EXPO_PUBLIC_SUPABASE_URL,
		supabaseAnonKey: process.env.EXPO_PUBLIC_SUPABASE_ANON_KEY,
		eas: { projectId: process.env.EAS_PROJECT_ID },
	},
});
```

---

## Root Layout (Providers)

```tsx
// app/_layout.tsx
import { useEffect } from "react";
import { Stack } from "expo-router";
import { StatusBar } from "expo-status-bar";
import { GestureHandlerRootView } from "react-native-gesture-handler";
import { QueryClientProvider } from "@tanstack/react-query";
import { queryClient } from "@/lib/query-client";
import { AuthProvider } from "@/lib/supabase/auth-context";
import * as SplashScreen from "expo-splash-screen";
import * as Notifications from "expo-notifications";
import { useNotificationObserver } from "@/hooks/use-notification-observer";

// Prevent splash screen from auto-hiding
SplashScreen.preventAutoHideAsync();

// Set notification handler BEFORE any component renders
Notifications.setNotificationHandler({
	handleNotification: async () => ({
		shouldShowAlert: true,
		shouldPlaySound: true,
		shouldSetBadge: true,
	}),
});

export default function RootLayout() {
	useNotificationObserver(); // Handle notification taps

	return (
		<GestureHandlerRootView style={{ flex: 1 }}>
			<QueryClientProvider client={queryClient}>
				<AuthProvider>
					<StatusBar style="auto" />
					<Stack screenOptions={{ headerShown: false }} />
				</AuthProvider>
			</QueryClientProvider>
		</GestureHandlerRootView>
	);
}
```

---

## Auth Navigation Pattern (Expo Router)

```tsx
// app/(auth)/_layout.tsx
import { Stack, Redirect } from "expo-router";
import { useAuth } from "@/lib/supabase/auth-context";

export default function AuthLayout() {
	const { session, isLoading } = useAuth();

	if (isLoading) return <SplashScreen />;
	if (session) return <Redirect href="/(app)/(tabs)/" />;

	return <Stack screenOptions={{ headerShown: false }} />;
}
```

```tsx
// app/(app)/_layout.tsx
import { Tabs, Redirect } from "expo-router";
import { useAuth } from "@/lib/supabase/auth-context";

export default function AppLayout() {
	const { session, isLoading } = useAuth();

	if (isLoading) return <SplashScreen />;
	if (!session) return <Redirect href="/(auth)/login" />;

	return (
		<Tabs
			screenOptions={{
				tabBarActiveTintColor: "#3b82f6",
				tabBarStyle: { paddingBottom: 4 },
				headerShown: false,
			}}
		>
			<Tabs.Screen
				name="(tabs)/index"
				options={{
					title: "Home",
					tabBarIcon: ({ color, size }) => (
						<Ionicons name="home-outline" size={size} color={color} />
					),
				}}
			/>
			<Tabs.Screen
				name="(tabs)/profile"
				options={{
					title: "Profile",
					tabBarIcon: ({ color, size }) => (
						<Ionicons name="person-outline" size={size} color={color} />
					),
				}}
			/>
		</Tabs>
	);
}
```

---

## Environment Variables

```bash
# .env.local (never commit)
EXPO_PUBLIC_SUPABASE_URL=https://xxxx.supabase.co
EXPO_PUBLIC_SUPABASE_ANON_KEY=eyJh...

# .env.example (commit — documents required vars)
EXPO_PUBLIC_SUPABASE_URL=
EXPO_PUBLIC_SUPABASE_ANON_KEY=
```

```ts
// src/lib/env.ts — validate at startup
import { z } from "zod";
import Constants from "expo-constants";

const envSchema = z.object({
	supabaseUrl: z.string().url(),
	supabaseAnonKey: z.string().min(1),
});

export const env = envSchema.parse({
	supabaseUrl: Constants.expoConfig?.extra?.supabaseUrl,
	supabaseAnonKey: Constants.expoConfig?.extra?.supabaseAnonKey,
});
```

---

## EAS Build Configuration

```json
// eas.json
{
	"cli": { "version": ">= 10.0.0" },
	"build": {
		"development": {
			"developmentClient": true,
			"distribution": "internal",
			"env": { "APP_ENV": "development" }
		},
		"preview": {
			"distribution": "internal",
			"env": { "APP_ENV": "preview" },
			"android": { "buildType": "apk" }
		},
		"production": {
			"env": { "APP_ENV": "production" },
			"autoIncrement": true
		}
	},
	"submit": {
		"production": {
			"ios": { "appleId": "your@email.com", "ascAppId": "123456789" },
			"android": { "serviceAccountKeyPath": "./google-service-account.json" }
		}
	}
}
```
