# NativeWind v4 + Design System

---

## Setup

NativeWind v4 uses Babel + Metro transforms to apply Tailwind
classes to React Native StyleSheet — not web CSS.

```ts
// tailwind.config.ts
import type { Config } from "tailwindcss";

export default {
	content: ["./app/**/*.{ts,tsx}", "./src/**/*.{ts,tsx}"],
	presets: [require("nativewind/preset")],
	theme: {
		extend: {
			colors: {
				// Brand tokens — single source of truth
				primary: {
					50: "#eff6ff",
					100: "#dbeafe",
					500: "#3b82f6",
					600: "#2563eb",
					700: "#1d4ed8",
				},
				// Semantic tokens
				surface: "#ffffff",
				"surface-2": "#f9fafb",
				border: "#e5e7eb",
				"text-primary": "#111827",
				"text-secondary": "#6b7280",
				"text-disabled": "#d1d5db",
				destructive: "#ef4444",
				success: "#22c55e",
				warning: "#f59e0b",
			},
			spacing: {
				// Design grid — multiples of 4
				"4.5": "18px",
			},
			borderRadius: {
				xl: "12px",
				"2xl": "16px",
				"3xl": "24px",
			},
		},
	},
} satisfies Config;
```

```ts
// babel.config.js
module.exports = function (api) {
	api.cache(true);
	return {
		presets: [
			["babel-preset-expo", { jsxImportSource: "nativewind" }],
			"nativewind/babel",
		],
	};
};
```

```ts
// metro.config.js
const { getDefaultConfig } = require("expo/metro-config");
const { withNativeWind } = require("nativewind/metro");

const config = getDefaultConfig(__dirname);
module.exports = withNativeWind(config, { input: "./global.css" });
```

---

## The `cn()` Utility for React Native

```ts
// src/utils/cn.ts
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

export function cn(...inputs: ClassValue[]) {
	return twMerge(clsx(inputs));
}
```

---

## Component Variants with `cva`

```tsx
// src/components/ui/button.tsx
import { Pressable, Text, ActivityIndicator, View } from "react-native";
import { cva, type VariantProps } from "class-variance-authority";
import { cn } from "@/utils/cn";

const buttonVariants = cva(
	// Base: always applied — min 44pt touch target
	"flex-row items-center justify-center rounded-xl min-h-[44px] px-4",
	{
		variants: {
			variant: {
				primary: "bg-primary-500 active:bg-primary-600",
				secondary: "bg-gray-100 active:bg-gray-200",
				destructive: "bg-red-500 active:bg-red-600",
				outline: "border border-gray-300 bg-transparent active:bg-gray-50",
				ghost: "bg-transparent active:bg-gray-100",
			},
			size: {
				sm: "min-h-[36px] px-3",
				md: "min-h-[44px] px-4",
				lg: "min-h-[52px] px-6",
				icon: "min-h-[44px] w-[44px] px-0",
			},
			fullWidth: {
				true: "w-full",
			},
		},
		defaultVariants: {
			variant: "primary",
			size: "md",
		},
	},
);

const textVariants = cva("font-semibold", {
	variants: {
		variant: {
			primary: "text-white",
			secondary: "text-gray-900",
			destructive: "text-white",
			outline: "text-gray-700",
			ghost: "text-gray-700",
		},
		size: {
			sm: "text-sm",
			md: "text-base",
			lg: "text-lg",
			icon: "text-base",
		},
	},
	defaultVariants: { variant: "primary", size: "md" },
});

interface ButtonProps extends VariantProps<typeof buttonVariants> {
	onPress?: () => void;
	children: React.ReactNode;
	isLoading?: boolean;
	disabled?: boolean;
	className?: string;
	accessibilityLabel?: string;
}

export function Button({
	variant,
	size,
	fullWidth,
	onPress,
	children,
	isLoading,
	disabled,
	className,
	accessibilityLabel,
}: ButtonProps) {
	const isDisabled = disabled || isLoading;

	return (
		<Pressable
			onPress={onPress}
			disabled={isDisabled}
			className={cn(
				buttonVariants({ variant, size, fullWidth }),
				isDisabled && "opacity-50",
				className,
			)}
			accessibilityRole="button"
			accessibilityLabel={accessibilityLabel}
			accessibilityState={{ disabled: isDisabled }}
		>
			{isLoading && (
				<ActivityIndicator
					color={
						variant === "primary" || variant === "destructive"
							? "#fff"
							: "#374151"
					}
					size="small"
					className="mr-2"
				/>
			)}
			<Text className={textVariants({ variant, size })}>{children}</Text>
		</Pressable>
	);
}
```

---

## Input Component

```tsx
// src/components/ui/input.tsx
import { TextInput, View, Text, type TextInputProps } from "react-native";
import { cn } from "@/utils/cn";

interface InputProps extends TextInputProps {
	label?: string;
	error?: string;
	hint?: string;
	leftIcon?: React.ReactNode;
	rightIcon?: React.ReactNode;
}

export function Input({
	label,
	error,
	hint,
	leftIcon,
	rightIcon,
	className,
	...props
}: InputProps) {
	return (
		<View className="space-y-1.5">
			{label && (
				<Text className="text-sm font-medium text-gray-700">{label}</Text>
			)}

			<View
				className={cn(
					"flex-row items-center rounded-xl border bg-white px-4",
					error ? "border-red-400" : "border-gray-300",
					props.editable === false && "bg-gray-50 opacity-60",
				)}
			>
				{leftIcon && <View className="mr-2">{leftIcon}</View>}
				<TextInput
					className={cn("flex-1 py-3 text-base text-gray-900", className)}
					placeholderTextColor="#9ca3af"
					autoCapitalize="none"
					{...props}
				/>
				{rightIcon && <View className="ml-2">{rightIcon}</View>}
			</View>

			{error && <Text className="text-sm text-red-500">{error}</Text>}
			{hint && !error && <Text className="text-sm text-gray-400">{hint}</Text>}
		</View>
	);
}
```

---

## Responsive Design (Dimensions-Based)

NativeWind doesn't have media queries in the web sense.
Use the `useWindowDimensions` hook for responsive layouts.

```ts
// src/utils/responsive.ts
import { useWindowDimensions } from "react-native";

export function useBreakpoint() {
	const { width } = useWindowDimensions();
	return {
		isSm: width >= 375, // Small phones
		isMd: width >= 430, // Large phones / small tablets
		isLg: width >= 768, // Tablets
		isXl: width >= 1024, // Large tablets / desktop (Expo Web)
		width,
	};
}

// Grid columns helper
export function useGridColumns(defaultCols = 1) {
	const { isLg, isMd } = useBreakpoint();
	if (isLg) return 3;
	if (isMd) return 2;
	return defaultCols;
}
```

```tsx
// Two-column grid on tablets, single on phones
export function ProjectGrid({ projects }: { projects: Project[] }) {
	const cols = useGridColumns(1);

	return (
		<FlashList
			data={projects}
			numColumns={cols}
			key={cols} // Re-mount when columns change (FlashList requirement)
			estimatedItemSize={120}
			renderItem={({ item }) => (
				<View className={cols === 2 ? "flex-1 mx-2" : "mx-4"}>
					<ProjectCard project={item} />
				</View>
			)}
		/>
	);
}
```

---

## Dark Mode

```tsx
// src/hooks/use-color-scheme.ts
import { useColorScheme as useRNColorScheme } from "react-native";

export function useColorScheme() {
	const scheme = useRNColorScheme();
	return {
		colorScheme: scheme ?? "light",
		isDark: scheme === "dark",
	};
}
```

```tsx
// NativeWind dark mode classes
<View className="bg-white dark:bg-gray-900">
	<Text className="text-gray-900 dark:text-white">Hello world</Text>
</View>
```

---

## Typography Scale

```
Screen title:     text-2xl font-bold text-gray-900
Section title:    text-lg font-semibold text-gray-900
Card title:       text-base font-semibold text-gray-900
Body:             text-base text-gray-700
Secondary body:   text-sm text-gray-500
Caption:          text-xs text-gray-400
Button label:     text-base font-semibold
Label:            text-sm font-medium text-gray-700
```

## Spacing Rules

```
Screen horizontal padding:   px-4 (16pt)
Screen top padding:          pt-4 (plus safe area)
Card padding:                p-4
Section gap:                 space-y-6 between sections
Item gap:                    space-y-3 within section
Icon size (nav):             24
Icon size (inline):          16-20
Touch target minimum:        min-h-[44px] min-w-[44px]
```
