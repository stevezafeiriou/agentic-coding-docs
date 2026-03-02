# Expo Push Notifications: End-to-End

---

## Architecture Overview

```
Device → Expo Push Service → APNs (iOS) / FCM (Android) → Device
                    ↑
              Your Supabase Edge Function
                    ↑
              Your app backend logic
```

---

## 1. Database Setup

```sql
-- Store push tokens per user per device
CREATE TABLE push_tokens (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  token       text NOT NULL UNIQUE,
  device_type text NOT NULL, -- 'ios' | 'android'
  is_active   boolean NOT NULL DEFAULT true,
  created_at  timestamptz DEFAULT now(),
  updated_at  timestamptz DEFAULT now()
);

CREATE INDEX push_tokens_user_id_idx ON push_tokens(user_id);
CREATE INDEX push_tokens_active_idx  ON push_tokens(user_id) WHERE is_active = true;

ALTER TABLE push_tokens ENABLE ROW LEVEL SECURITY;

-- Users manage only their own tokens
CREATE POLICY "users manage own push tokens"
  ON push_tokens FOR ALL
  USING (user_id = auth.uid())
  WITH CHECK (user_id = auth.uid());

-- Notifications inbox
CREATE TABLE notifications (
  id          uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  title       text NOT NULL,
  body        text,
  data        jsonb DEFAULT '{}',
  is_read     boolean NOT NULL DEFAULT false,
  created_at  timestamptz DEFAULT now()
);

CREATE INDEX notifications_user_id_idx ON notifications(user_id);
CREATE INDEX notifications_unread_idx  ON notifications(user_id) WHERE is_read = false;

ALTER TABLE notifications ENABLE ROW LEVEL SECURITY;

CREATE POLICY "users see own notifications"
  ON notifications FOR SELECT USING (user_id = auth.uid());

CREATE POLICY "users update own notifications"
  ON notifications FOR UPDATE USING (user_id = auth.uid());
```

---

## 2. Permission Request & Token Registration

```ts
// src/lib/notifications/register.ts
import * as Notifications from "expo-notifications";
import * as Device from "expo-device";
import { Platform } from "react-native";
import { supabase } from "@/lib/supabase/client";

export async function registerForPushNotifications(): Promise<string | null> {
	// Expo Go on simulator won't have a real token — handle gracefully
	if (!Device.isDevice) {
		console.warn("Push notifications require a physical device");
		return null;
	}

	// Check existing permission
	const { status: existingStatus } = await Notifications.getPermissionsAsync();
	let finalStatus = existingStatus;

	if (existingStatus !== "granted") {
		const { status } = await Notifications.requestPermissionsAsync();
		finalStatus = status;
	}

	if (finalStatus !== "granted") {
		return null; // User denied — respect it, don't re-prompt
	}

	// Android requires notification channel
	if (Platform.OS === "android") {
		await Notifications.setNotificationChannelAsync("default", {
			name: "Default",
			importance: Notifications.AndroidImportance.MAX,
			vibrationPattern: [0, 250, 250, 250],
			lightColor: "#3b82f6",
		});
	}

	try {
		const tokenData = await Notifications.getExpoPushTokenAsync({
			projectId: Constants.expoConfig?.extra?.eas?.projectId,
		});
		return tokenData.data;
	} catch (err) {
		console.error("Failed to get push token:", err);
		return null;
	}
}

export async function savePushToken(token: string): Promise<void> {
	const {
		data: { user },
	} = await supabase.auth.getUser();
	if (!user) return;

	// Upsert — handles re-registration after reinstall
	const { error } = await supabase.from("push_tokens").upsert(
		{
			user_id: user.id,
			token,
			device_type: Platform.OS,
			is_active: true,
			updated_at: new Date().toISOString(),
		},
		{ onConflict: "token" },
	);

	if (error) console.error("Failed to save push token:", error);
}

export async function deactivatePushToken(token: string): Promise<void> {
	await supabase
		.from("push_tokens")
		.update({ is_active: false })
		.eq("token", token);
}
```

---

## 3. Notification Observer Hook (App Root)

```ts
// src/hooks/use-notification-observer.ts
import { useEffect, useRef } from "react";
import { AppState } from "react-native";
import * as Notifications from "expo-notifications";
import { useRouter } from "expo-router";
import { useQueryClient } from "@tanstack/react-query";
import {
	registerForPushNotifications,
	savePushToken,
} from "@/lib/notifications/register";
import { useAuth } from "@/lib/supabase/auth-context";
import { queryKeys } from "@/lib/queries/keys";

export function useNotificationObserver() {
	const router = useRouter();
	const queryClient = useQueryClient();
	const { user } = useAuth();
	const notificationListener = useRef<Notifications.Subscription>();
	const responseListener = useRef<Notifications.Subscription>();

	useEffect(() => {
		if (!user) return;

		// Register token on login
		registerForPushNotifications().then((token) => {
			if (token) savePushToken(token);
		});

		// Foreground: notification received while app is open
		notificationListener.current =
			Notifications.addNotificationReceivedListener((notification) => {
				// Invalidate notifications query to update badge
				queryClient.invalidateQueries({
					queryKey: queryKeys.notifications.unread(),
				});

				// Handle specific notification types
				const data = notification.request.content.data;
				if (data?.type === "task_assigned") {
					queryClient.invalidateQueries({
						queryKey: queryKeys.tasks.byProject(data.project_id as string),
					});
				}
			});

		// Background/quit: user taps notification
		responseListener.current =
			Notifications.addNotificationResponseReceivedListener((response) => {
				const data = response.notification.request.content.data;

				// Navigate based on notification type
				if (data?.type === "task_assigned" && data.task_id) {
					router.push(`/(app)/tasks/${data.task_id}`);
				} else if (data?.type === "new_message" && data.channel_id) {
					router.push(`/(app)/channels/${data.channel_id}`);
				}
			});

		return () => {
			notificationListener.current?.remove();
			responseListener.current?.remove();
		};
	}, [user, router, queryClient]);
}
```

---

## 4. Supabase Edge Function: Send Notifications

```ts
// supabase/functions/send-notification/index.ts
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";
import { corsHeaders } from "../_shared/cors.ts";

interface NotificationPayload {
	userId: string;
	title: string;
	body: string;
	data?: Record<string, unknown>;
}

const EXPO_PUSH_URL = "https://exp.host/--/api/v2/push/send";

Deno.serve(async (req) => {
	if (req.method === "OPTIONS")
		return new Response("ok", { headers: corsHeaders });

	const supabase = createClient(
		Deno.env.get("SUPABASE_URL")!,
		Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!,
	);

	const payload: NotificationPayload = await req.json();

	// 1. Get active push tokens for the user
	const { data: tokens, error } = await supabase
		.from("push_tokens")
		.select("token")
		.eq("user_id", payload.userId)
		.eq("is_active", true);

	if (error || !tokens?.length) {
		return new Response(JSON.stringify({ sent: 0 }), {
			headers: { ...corsHeaders, "Content-Type": "application/json" },
		});
	}

	// 2. Store notification in inbox
	await supabase.from("notifications").insert({
		user_id: payload.userId,
		title: payload.title,
		body: payload.body,
		data: payload.data ?? {},
	});

	// 3. Send via Expo Push API (batching — max 100 per request)
	const messages = tokens.map(({ token }) => ({
		to: token,
		title: payload.title,
		body: payload.body,
		data: payload.data ?? {},
		sound: "default",
		badge: 1,
	}));

	const response = await fetch(EXPO_PUSH_URL, {
		method: "POST",
		headers: { "Content-Type": "application/json", Accept: "application/json" },
		body: JSON.stringify(messages),
	});

	const result = await response.json();

	// 4. Deactivate invalid tokens
	const invalidTokens: string[] = [];
	result.data?.forEach(
		(ticket: { status: string; details?: { error: string } }, i: number) => {
			if (
				ticket.status === "error" &&
				ticket.details?.error === "DeviceNotRegistered"
			) {
				invalidTokens.push(tokens[i].token);
			}
		},
	);

	if (invalidTokens.length > 0) {
		await supabase
			.from("push_tokens")
			.update({ is_active: false })
			.in("token", invalidTokens);
	}

	return new Response(
		JSON.stringify({ sent: messages.length - invalidTokens.length }),
		{ headers: { ...corsHeaders, "Content-Type": "application/json" } },
	);
});
```

---

## 5. Notification Inbox UI

```tsx
// src/hooks/use-notifications.ts
export function useNotifications() {
	return useQuery({
		queryKey: queryKeys.notifications.all(),
		queryFn: async () => {
			const { data, error } = await supabase
				.from("notifications")
				.select("*")
				.order("created_at", { ascending: false })
				.limit(50); // Paginate — never fetch unbounded
			if (error) throw new Error(error.message);
			return data;
		},
		staleTime: 1000 * 30, // 30s — notifications can be slightly stale
	});
}

export function useMarkNotificationRead() {
	const queryClient = useQueryClient();

	return useMutation({
		mutationFn: async (id: string) => {
			const { error } = await supabase
				.from("notifications")
				.update({ is_read: true })
				.eq("id", id);
			if (error) throw error;
		},
		onMutate: async (id) => {
			// Optimistic update
			queryClient.setQueryData(
				queryKeys.notifications.all(),
				(old: Notification[] = []) =>
					old.map((n) => (n.id === id ? { ...n, is_read: true } : n)),
			);
		},
		onSettled: () => {
			queryClient.invalidateQueries({
				queryKey: queryKeys.notifications.all(),
			});
		},
	});
}
```
