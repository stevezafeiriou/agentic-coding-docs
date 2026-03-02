# React Native Component Patterns

---

## Component Template

```tsx
// src/components/features/task-card.tsx
import { Pressable, View, Text } from "react-native";
import { useRouter } from "expo-router";
import type { Tables } from "@/lib/supabase/types";

type Task = Tables<"tasks">;

interface TaskCardProps {
	task: Task;
	onComplete?: (id: string) => void;
}

export function TaskCard({ task, onComplete }: TaskCardProps) {
	const router = useRouter();

	return (
		<Pressable
			className="rounded-xl bg-white p-4 shadow-sm active:opacity-70"
			onPress={() => router.push(`/(app)/tasks/${task.id}`)}
			// Minimum 44pt touch target (Apple HIG)
			style={{ minHeight: 44 }}
			accessibilityRole="button"
			accessibilityLabel={`Task: ${task.title}`}
		>
			<View className="flex-row items-center justify-between">
				<View className="flex-1 mr-3">
					<Text
						className="text-base font-medium text-gray-900"
						numberOfLines={1}
					>
						{task.title}
					</Text>
					{task.description && (
						<Text className="mt-1 text-sm text-gray-500" numberOfLines={2}>
							{task.description}
						</Text>
					)}
				</View>

				{onComplete && (
					<Pressable
						onPress={() => onComplete(task.id)}
						hitSlop={8} // Expand touch area without changing visual
						className="h-6 w-6 items-center justify-center rounded-full border-2 border-blue-500"
						accessibilityRole="checkbox"
						accessibilityState={{ checked: task.is_complete }}
					>
						{task.is_complete && (
							<View className="h-3 w-3 rounded-full bg-blue-500" />
						)}
					</Pressable>
				)}
			</View>
		</Pressable>
	);
}
```

---

## FlashList — Replace All FlatList Usage

FlashList is 10x faster than FlatList. Use it for every list of 10+ items.

```tsx
// src/components/features/task-list.tsx
import { FlashList } from "@shopify/flash-list";
import { View, Text, RefreshControl } from "react-native";
import { useCallback } from "react";
import { TaskCard } from "./task-card";
import type { Tables } from "@/lib/supabase/types";

type Task = Tables<"tasks">;

interface TaskListProps {
	tasks: Task[];
	isRefreshing?: boolean;
	onRefresh?: () => void;
	onComplete?: (id: string) => void;
	ListHeaderComponent?: React.ReactElement;
}

export function TaskList({
	tasks,
	isRefreshing = false,
	onRefresh,
	onComplete,
	ListHeaderComponent,
}: TaskListProps) {
	// MUST be stable — use useCallback to prevent re-renders
	const renderItem = useCallback(
		({ item }: { item: Task }) => (
			<TaskCard task={item} onComplete={onComplete} />
		),
		[onComplete],
	);

	const keyExtractor = useCallback((item: Task) => item.id, []);

	const ItemSeparator = useCallback(() => <View className="h-2" />, []);

	if (tasks.length === 0) {
		return (
			<View className="flex-1 items-center justify-center p-8">
				<Text className="text-base text-gray-400">No tasks yet</Text>
			</View>
		);
	}

	return (
		<FlashList
			data={tasks}
			renderItem={renderItem}
			keyExtractor={keyExtractor}
			estimatedItemSize={80} // Required by FlashList — estimate your item height
			ItemSeparatorComponent={ItemSeparator}
			ListHeaderComponent={ListHeaderComponent}
			contentContainerStyle={{ padding: 16 }}
			refreshControl={
				onRefresh ? (
					<RefreshControl refreshing={isRefreshing} onRefresh={onRefresh} />
				) : undefined
			}
			showsVerticalScrollIndicator={false}
		/>
	);
}
```

---

## Keyboard Handling

```tsx
// src/components/ui/keyboard-aware-view.tsx
import {
	KeyboardAvoidingView,
	Platform,
	ScrollView,
	TouchableWithoutFeedback,
	Keyboard,
	type ViewStyle,
} from "react-native";

interface KeyboardAwareViewProps {
	children: React.ReactNode;
	style?: ViewStyle;
	scrollEnabled?: boolean;
}

export function KeyboardAwareView({
	children,
	style,
	scrollEnabled = true,
}: KeyboardAwareViewProps) {
	return (
		<KeyboardAvoidingView
			style={[{ flex: 1 }, style]}
			behavior={Platform.OS === "ios" ? "padding" : "height"}
			keyboardVerticalOffset={Platform.OS === "ios" ? 0 : 20}
		>
			<TouchableWithoutFeedback onPress={Keyboard.dismiss} accessible={false}>
				{scrollEnabled ? (
					<ScrollView
						keyboardShouldPersistTaps="handled"
						showsVerticalScrollIndicator={false}
					>
						{children}
					</ScrollView>
				) : (
					<>{children}</>
				)}
			</TouchableWithoutFeedback>
		</KeyboardAvoidingView>
	);
}
```

---

## Safe Area Handling

```tsx
// Always use useSafeAreaInsets for custom headers/footers
// Never hardcode top/bottom padding

import { useSafeAreaInsets } from "react-native-safe-area-context";
import { View } from "react-native";

export function ScreenHeader({ title }: { title: string }) {
	const insets = useSafeAreaInsets();

	return (
		<View
			className="bg-white border-b border-gray-100 px-4 pb-3"
			style={{ paddingTop: insets.top + 8 }} // Safe area + design spacing
		>
			<Text className="text-xl font-semibold text-gray-900">{title}</Text>
		</View>
	);
}

// For tab bar content — add bottom safe area
export function ScreenContent({ children }: { children: React.ReactNode }) {
	const insets = useSafeAreaInsets();

	return (
		<View className="flex-1" style={{ paddingBottom: insets.bottom }}>
			{children}
		</View>
	);
}
```

---

## Platform-Specific Patterns

```tsx
import { Platform, StyleSheet } from "react-native";

// Shadow — iOS uses shadow props, Android uses elevation
export const cardShadow = Platform.select({
	ios: {
		shadowColor: "#000",
		shadowOffset: { width: 0, height: 1 },
		shadowOpacity: 0.08,
		shadowRadius: 4,
	},
	android: {
		elevation: 2,
	},
});

// Platform-specific component variant
export function Divider() {
	return (
		<View
			className={
				Platform.OS === "ios"
					? "h-px bg-gray-200 ml-4" // iOS: inset divider
					: "h-px bg-gray-200" // Android: full-width
			}
		/>
	);
}
```

---

## Reanimated: Smooth Animations on UI Thread

```tsx
// src/components/ui/animated-pressable.tsx
// Runs entirely on UI thread — never blocks JS

import Animated, {
	useSharedValue,
	useAnimatedStyle,
	withSpring,
} from "react-native-reanimated";
import { Pressable } from "react-native";

const AnimatedPressable = Animated.createAnimatedComponent(Pressable);

export function ScalePressable({
	children,
	onPress,
	className,
}: {
	children: React.ReactNode;
	onPress?: () => void;
	className?: string;
}) {
	const scale = useSharedValue(1);

	const animatedStyle = useAnimatedStyle(() => ({
		transform: [{ scale: scale.value }],
	}));

	return (
		<AnimatedPressable
			style={animatedStyle}
			className={className}
			onPressIn={() => {
				scale.value = withSpring(0.96);
			}}
			onPressOut={() => {
				scale.value = withSpring(1);
			}}
			onPress={onPress}
		>
			{children}
		</AnimatedPressable>
	);
}
```

---

## Loading & Error State Components

```tsx
// src/components/ui/screen-states.tsx
import { View, Text, ActivityIndicator, Pressable } from "react-native";
import { Ionicons } from "@expo/vector-icons";

export function LoadingScreen() {
	return (
		<View className="flex-1 items-center justify-center bg-white">
			<ActivityIndicator size="large" color="#3b82f6" />
		</View>
	);
}

export function ErrorScreen({
	message = "Something went wrong",
	onRetry,
}: {
	message?: string;
	onRetry?: () => void;
}) {
	return (
		<View className="flex-1 items-center justify-center bg-white p-6">
			<Ionicons name="alert-circle-outline" size={48} color="#ef4444" />
			<Text className="mt-4 text-center text-base font-medium text-gray-900">
				{message}
			</Text>
			{onRetry && (
				<Pressable
					onPress={onRetry}
					className="mt-4 rounded-lg bg-blue-500 px-6 py-3"
				>
					<Text className="font-medium text-white">Try again</Text>
				</Pressable>
			)}
		</View>
	);
}

export function EmptyScreen({
	icon,
	title,
	description,
	action,
}: {
	icon: keyof typeof Ionicons.glyphMap;
	title: string;
	description: string;
	action?: React.ReactNode;
}) {
	return (
		<View className="flex-1 items-center justify-center bg-white p-8">
			<View className="h-16 w-16 items-center justify-center rounded-full bg-gray-100">
				<Ionicons name={icon} size={32} color="#9ca3af" />
			</View>
			<Text className="mt-4 text-center text-lg font-semibold text-gray-900">
				{title}
			</Text>
			<Text className="mt-2 text-center text-sm text-gray-500">
				{description}
			</Text>
			{action && <View className="mt-6">{action}</View>}
		</View>
	);
}

// Skeleton loading — match the shape of real content
export function CardSkeleton() {
	return (
		<View className="rounded-xl bg-white p-4 shadow-sm">
			<View className="h-4 w-3/4 animate-pulse rounded bg-gray-200" />
			<View className="mt-2 h-3 w-1/2 animate-pulse rounded bg-gray-100" />
		</View>
	);
}
```

---

## Form Pattern

```tsx
// src/components/features/create-task-form.tsx
import {
	View,
	Text,
	TextInput,
	Pressable,
	ActivityIndicator,
} from "react-native";
import { useForm, Controller } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";
import { useCreateTask } from "@/hooks/use-tasks";

const schema = z.object({
	title: z.string().min(1, "Title is required").max(100, "Title too long"),
	description: z.string().max(500).optional(),
});
type FormData = z.infer<typeof schema>;

export function CreateTaskForm({ onSuccess }: { onSuccess?: () => void }) {
	const { mutate, isPending } = useCreateTask();

	const {
		control,
		handleSubmit,
		formState: { errors },
		reset,
	} = useForm<FormData>({
		resolver: zodResolver(schema),
		defaultValues: { title: "", description: "" },
	});

	function onSubmit(data: FormData) {
		mutate(data, {
			onSuccess: () => {
				reset();
				onSuccess?.();
			},
		});
	}

	return (
		<View className="space-y-4 p-4">
			<Controller
				control={control}
				name="title"
				render={({ field: { onChange, value, onBlur } }) => (
					<View className="space-y-1">
						<Text className="text-sm font-medium text-gray-700">Title</Text>
						<TextInput
							value={value}
							onChangeText={onChange}
							onBlur={onBlur}
							placeholder="Task title"
							placeholderTextColor="#9ca3af"
							returnKeyType="next"
							className={`rounded-lg border px-4 py-3 text-base text-gray-900 ${
								errors.title ? "border-red-400" : "border-gray-300"
							}`}
						/>
						{errors.title && (
							<Text className="text-sm text-red-500">
								{errors.title.message}
							</Text>
						)}
					</View>
				)}
			/>

			<Pressable
				onPress={handleSubmit(onSubmit)}
				disabled={isPending}
				className="rounded-lg bg-blue-500 py-3 items-center disabled:opacity-50"
			>
				{isPending ? (
					<ActivityIndicator color="#fff" size="small" />
				) : (
					<Text className="font-semibold text-white">Create Task</Text>
				)}
			</Pressable>
		</View>
	);
}
```
