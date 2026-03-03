# Expo Router Complete Reference

Expo Router is a file-based routing framework for React Native and Expo applications. It brings the file-system routing paradigm from web frameworks like Next.js to mobile development.

## File Conventions

### Route Files

| File | Purpose |
|------|---------|
| `index.tsx` | Default route for a directory (e.g., `app/index.tsx` → `/`) |
| `[param].tsx` | Dynamic segment (e.g., `[id].tsx` → `/123`) |
| `[...rest].tsx` | Catch-all route (e.g., `[...slug].tsx` → `/a/b/c`) |
| `_layout.tsx` | Layout wrapper for sibling and child routes |
| `+not-found.tsx` | 404 fallback page |
| `+html.tsx` | Custom HTML wrapper (web only) |

### Special Naming

| Pattern | Effect |
|---------|--------|
| `(group)` | Route group - organizes files without affecting URL |
| `_hidden.tsx` | Files prefixed with `_` are excluded from routing |
| `+api.ts` | API route (server function) |

## Route Groups

Route groups organize routes without adding segments to the URL:

```
app/
├── (tabs)/
│   ├── _layout.tsx      # Tab layout
│   ├── index.tsx        # / (home tab)
│   ├── settings.tsx     # /settings
│   └── profile.tsx      # /profile
├── (auth)/
│   ├── _layout.tsx      # Auth layout (no tabs)
│   ├── login.tsx        # /login
│   └── register.tsx     # /register
└── _layout.tsx          # Root layout
```

### Common Route Group Patterns

**Tab Navigation:**
```
(tabs)/
├── _layout.tsx          # Tabs component
├── index.tsx            # First tab
└── explore.tsx          # Second tab
```

**Authentication Flow:**
```
(auth)/
├── _layout.tsx          # Stack without header
├── login.tsx
├── register.tsx
└── forgot-password.tsx
```

**Modal Routes:**
```
app/
├── (main)/
│   ├── _layout.tsx      # Stack layout
│   └── index.tsx
├── modal.tsx            # Opens as modal over (main)
└── _layout.tsx          # Root with modal presentation
```

## Layout Files

### Root Layout (`app/_layout.tsx`)

```typescript
import { Stack } from 'expo-router'
import { TamaguiProvider } from 'tamagui'
import config from '../tamagui.config'

export default function RootLayout() {
  return (
    <TamaguiProvider config={config} defaultTheme="light">
      <Stack>
        <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
        <Stack.Screen name="modal" options={{ presentation: 'modal' }} />
      </Stack>
    </TamaguiProvider>
  )
}
```

### Tab Layout (`app/(tabs)/_layout.tsx`)

```typescript
import { Tabs } from 'expo-router'
import { Home, Settings, User } from '@tamagui/lucide-icons'

export default function TabLayout() {
  return (
    <Tabs
      screenOptions={{
        tabBarActiveTintColor: '#007AFF',
        headerShown: false,
      }}
    >
      <Tabs.Screen
        name="index"
        options={{
          title: 'Home',
          tabBarIcon: ({ color }) => <Home color={color} />,
        }}
      />
      <Tabs.Screen
        name="settings"
        options={{
          title: 'Settings',
          tabBarIcon: ({ color }) => <Settings color={color} />,
        }}
      />
      <Tabs.Screen
        name="profile"
        options={{
          title: 'Profile',
          tabBarIcon: ({ color }) => <User color={color} />,
        }}
      />
    </Tabs>
  )
}
```

### Stack Layout Options

```typescript
import { Stack } from 'expo-router'

export default function StackLayout() {
  return (
    <Stack
      screenOptions={{
        headerStyle: { backgroundColor: '#f5f5f5' },
        headerTintColor: '#000',
        headerTitleStyle: { fontWeight: 'bold' },
        contentStyle: { backgroundColor: '#fff' },
        animation: 'slide_from_right',
      }}
    >
      <Stack.Screen name="index" options={{ title: 'Home' }} />
      <Stack.Screen
        name="details"
        options={{
          title: 'Details',
          presentation: 'card',
        }}
      />
      <Stack.Screen
        name="settings"
        options={{
          presentation: 'modal',
          animation: 'slide_from_bottom',
        }}
      />
    </Stack>
  )
}
```

## Navigation

### Link Component

```typescript
import { Link } from 'expo-router'

// Basic link
<Link href="/settings">Go to Settings</Link>

// Link with params
<Link href="/users/123">View User</Link>
<Link href={{ pathname: '/users/[id]', params: { id: '123' } }}>
  View User
</Link>

// Replace instead of push
<Link href="/home" replace>Go Home</Link>

// Link as child (pass styles to child)
<Link href="/profile" asChild>
  <Button>View Profile</Button>
</Link>
```

### useRouter Hook

```typescript
import { useRouter } from 'expo-router'

function NavigationExample() {
  const router = useRouter()

  return (
    <YStack gap="$2">
      {/* Push new screen */}
      <Button onPress={() => router.push('/settings')}>
        Go to Settings
      </Button>

      {/* Push with params */}
      <Button onPress={() => router.push({
        pathname: '/users/[id]',
        params: { id: '123' }
      })}>
        View User
      </Button>

      {/* Replace current screen */}
      <Button onPress={() => router.replace('/home')}>
        Replace with Home
      </Button>

      {/* Go back */}
      <Button onPress={() => router.back()}>
        Go Back
      </Button>

      {/* Can go back check */}
      <Button
        onPress={() => router.canGoBack() && router.back()}
        disabled={!router.canGoBack()}
      >
        Safe Back
      </Button>

      {/* Navigate (push or replace based on history) */}
      <Button onPress={() => router.navigate('/search')}>
        Navigate to Search
      </Button>

      {/* Dismiss all modals */}
      <Button onPress={() => router.dismissAll()}>
        Dismiss All
      </Button>
    </YStack>
  )
}
```

### router Methods Reference

| Method | Description |
|--------|-------------|
| `push(href)` | Navigate to route, add to history |
| `replace(href)` | Replace current route in history |
| `back()` | Go back one screen |
| `canGoBack()` | Check if back navigation is possible |
| `navigate(href)` | Navigate smartly (like web history) |
| `dismiss()` | Dismiss current modal |
| `dismissAll()` | Dismiss all modals |
| `setParams(params)` | Update current route params |

## Route Parameters

### Reading Parameters

```typescript
import { useLocalSearchParams, useGlobalSearchParams } from 'expo-router'

// In app/users/[id].tsx
function UserScreen() {
  // Get params from current route only
  const { id } = useLocalSearchParams<{ id: string }>()

  // Get params from entire URL (including parent routes)
  const globalParams = useGlobalSearchParams()

  return <Text>User ID: {id}</Text>
}
```

### Typed Parameters

```typescript
// Define param types
type UserParams = {
  id: string
  tab?: 'posts' | 'likes' | 'comments'
}

function UserScreen() {
  const { id, tab = 'posts' } = useLocalSearchParams<UserParams>()

  return (
    <YStack>
      <Text>User: {id}</Text>
      <Text>Tab: {tab}</Text>
    </YStack>
  )
}
```

### Catch-All Routes

```typescript
// app/docs/[...slug].tsx
import { useLocalSearchParams } from 'expo-router'

function DocsPage() {
  // slug is array: ['guides', 'getting-started']
  const { slug } = useLocalSearchParams<{ slug: string[] }>()

  const path = slug?.join('/') ?? ''
  return <Text>Doc path: {path}</Text>
}
```

## Dynamic Routes

### Single Dynamic Segment

```
app/
└── users/
    └── [id].tsx        # /users/123, /users/abc
```

```typescript
// app/users/[id].tsx
import { useLocalSearchParams } from 'expo-router'

export default function UserScreen() {
  const { id } = useLocalSearchParams<{ id: string }>()
  return <Text>User ID: {id}</Text>
}
```

### Multiple Dynamic Segments

```
app/
└── [category]/
    └── [product].tsx   # /electronics/laptop, /books/novel
```

```typescript
// app/[category]/[product].tsx
import { useLocalSearchParams } from 'expo-router'

export default function ProductScreen() {
  const { category, product } = useLocalSearchParams<{
    category: string
    product: string
  }>()

  return (
    <YStack>
      <Text>Category: {category}</Text>
      <Text>Product: {product}</Text>
    </YStack>
  )
}
```

### Optional Catch-All

```
app/
└── [[...path]].tsx     # /, /a, /a/b/c (matches root too)
```

## Tab Navigation

### Basic Tabs

```typescript
// app/(tabs)/_layout.tsx
import { Tabs } from 'expo-router'

export default function TabLayout() {
  return (
    <Tabs>
      <Tabs.Screen name="index" options={{ title: 'Home' }} />
      <Tabs.Screen name="explore" options={{ title: 'Explore' }} />
    </Tabs>
  )
}
```

### Custom Tab Bar

```typescript
import { Tabs } from 'expo-router'
import { XStack, Text } from 'tamagui'

function CustomTabBar({ state, descriptors, navigation }) {
  return (
    <XStack backgroundColor="$background" paddingBottom="$4">
      {state.routes.map((route, index) => {
        const { options } = descriptors[route.key]
        const isFocused = state.index === index

        const onPress = () => {
          const event = navigation.emit({
            type: 'tabPress',
            target: route.key,
          })
          if (!isFocused && !event.defaultPrevented) {
            navigation.navigate(route.name)
          }
        }

        return (
          <XStack
            key={route.key}
            flex={1}
            alignItems="center"
            padding="$2"
            onPress={onPress}
          >
            <Text color={isFocused ? '$primary' : '$color'}>
              {options.title || route.name}
            </Text>
          </XStack>
        )
      })}
    </XStack>
  )
}

export default function TabLayout() {
  return <Tabs tabBar={props => <CustomTabBar {...props} />} />
}
```

### Hide Tab on Specific Screens

```typescript
import { Tabs } from 'expo-router'

export default function TabLayout() {
  return (
    <Tabs>
      <Tabs.Screen name="index" />
      <Tabs.Screen
        name="hidden"
        options={{
          href: null,  // Hide from tab bar but keep route
        }}
      />
    </Tabs>
  )
}
```

## Deep Linking

### URL Scheme Configuration

```json
// app.json
{
  "expo": {
    "scheme": "myapp",
    "web": {
      "bundler": "metro"
    }
  }
}
```

### Link Formats

| Platform | Format |
|----------|--------|
| iOS/Android | `myapp://path/to/screen` |
| Web | `https://myapp.com/path/to/screen` |
| Expo Go | `exp://192.168.1.x:8081/--/path` |

### Handling Deep Links

```typescript
// app/_layout.tsx
import { useEffect } from 'react'
import { Linking } from 'react-native'
import { useRouter } from 'expo-router'

export default function RootLayout() {
  const router = useRouter()

  useEffect(() => {
    // Handle initial URL
    Linking.getInitialURL().then((url) => {
      if (url) handleDeepLink(url)
    })

    // Handle URL changes
    const subscription = Linking.addEventListener('url', ({ url }) => {
      handleDeepLink(url)
    })

    return () => subscription.remove()
  }, [])

  function handleDeepLink(url: string) {
    // Parse and navigate
    const path = url.replace(/.*:\/\//, '/')
    router.push(path)
  }

  return <Stack />
}
```

### Universal Links (iOS) / App Links (Android)

```json
// app.json
{
  "expo": {
    "ios": {
      "associatedDomains": ["applinks:myapp.com"]
    },
    "android": {
      "intentFilters": [
        {
          "action": "VIEW",
          "autoVerify": true,
          "data": [
            {
              "scheme": "https",
              "host": "myapp.com",
              "pathPrefix": "/"
            }
          ],
          "category": ["BROWSABLE", "DEFAULT"]
        }
      ]
    }
  }
}
```

## Error Handling

### Error Boundary Export

```typescript
// app/users/[id].tsx
import { ErrorBoundary } from 'expo-router'
import { YStack, Text, Button } from 'tamagui'

// Named export for error boundary
export function ErrorBoundary({ error, retry }) {
  return (
    <YStack flex={1} alignItems="center" justifyContent="center" padding="$4">
      <Text color="$red10" marginBottom="$4">
        {error.message}
      </Text>
      <Button onPress={retry}>Try Again</Button>
    </YStack>
  )
}

export default function UserScreen() {
  // Component that might throw
}
```

### Not Found Handling

```typescript
// app/+not-found.tsx
import { Link, Stack } from 'expo-router'
import { YStack, Text, Button } from 'tamagui'

export default function NotFoundScreen() {
  return (
    <>
      <Stack.Screen options={{ title: 'Not Found' }} />
      <YStack flex={1} alignItems="center" justifyContent="center" padding="$4">
        <Text fontSize="$6" marginBottom="$4">
          Page Not Found
        </Text>
        <Link href="/" asChild>
          <Button>Go Home</Button>
        </Link>
      </YStack>
    </>
  )
}
```

## Typed Routes (TypeScript)

### Enable Typed Routes

```json
// app.json
{
  "expo": {
    "experiments": {
      "typedRoutes": true
    }
  }
}
```

### Generate Types

```bash
bunx expo customize tsconfig.json
```

### Using Typed Routes

```typescript
import { Link, useRouter } from 'expo-router'

// TypeScript will validate these routes exist
<Link href="/settings">Settings</Link>
<Link href="/users/123">User</Link>

const router = useRouter()
router.push('/valid-route')  // OK
router.push('/invalid')      // Type error if route doesn't exist
```

## Protected Routes

### Authentication Pattern

```typescript
// app/_layout.tsx
import { Redirect, Stack } from 'expo-router'
import { useAuth } from '../hooks/useAuth'

export default function RootLayout() {
  return (
    <Stack>
      <Stack.Screen name="(auth)" options={{ headerShown: false }} />
      <Stack.Screen name="(app)" options={{ headerShown: false }} />
    </Stack>
  )
}

// app/(app)/_layout.tsx
import { Redirect, Stack } from 'expo-router'
import { useAuth } from '../../hooks/useAuth'

export default function AppLayout() {
  const { isAuthenticated, isLoading } = useAuth()

  if (isLoading) {
    return <LoadingScreen />
  }

  if (!isAuthenticated) {
    return <Redirect href="/login" />
  }

  return <Stack />
}
```

### Per-Screen Protection

```typescript
// app/(app)/settings.tsx
import { Redirect } from 'expo-router'
import { useAuth } from '../../hooks/useAuth'

export default function SettingsScreen() {
  const { user } = useAuth()

  if (!user?.isAdmin) {
    return <Redirect href="/" />
  }

  return <AdminSettings />
}
```

## Screen Options

### Static Options

```typescript
// app/about.tsx
import { Stack } from 'expo-router'

export default function AboutScreen() {
  return (
    <>
      <Stack.Screen
        options={{
          title: 'About',
          headerStyle: { backgroundColor: '#f5f5f5' },
          headerTintColor: '#000',
          headerBackTitle: 'Back',
        }}
      />
      <YStack padding="$4">
        <Text>About content</Text>
      </YStack>
    </>
  )
}
```

### Dynamic Options

```typescript
import { Stack, useLocalSearchParams } from 'expo-router'

export default function UserScreen() {
  const { id } = useLocalSearchParams<{ id: string }>()
  const [user, setUser] = useState(null)

  return (
    <>
      <Stack.Screen
        options={{
          title: user?.name || 'Loading...',
          headerRight: () => (
            <Button size="$2" onPress={() => {}}>
              Edit
            </Button>
          ),
        }}
      />
      <UserProfile user={user} />
    </>
  )
}
```

## Segment Configuration

### Unstable Settings (Advanced)

```typescript
// app/(tabs)/settings.tsx
export const unstable_settings = {
  // Ensure this tab always shows its initial route
  initialRouteName: 'index',
}

export default function SettingsScreen() {
  return <Settings />
}
```

## Navigation Events

### Focus/Blur Events

```typescript
import { useFocusEffect } from 'expo-router'
import { useCallback } from 'react'

export default function Screen() {
  useFocusEffect(
    useCallback(() => {
      // Screen focused
      console.log('Screen focused')

      return () => {
        // Screen blurred
        console.log('Screen blurred')
      }
    }, [])
  )

  return <View />
}
```

### Prevent Navigation

```typescript
import { useNavigation } from 'expo-router'
import { useEffect } from 'react'

export default function FormScreen() {
  const navigation = useNavigation()
  const [hasUnsavedChanges, setHasUnsavedChanges] = useState(false)

  useEffect(() => {
    const unsubscribe = navigation.addListener('beforeRemove', (e) => {
      if (!hasUnsavedChanges) return

      e.preventDefault()

      Alert.alert(
        'Discard changes?',
        'You have unsaved changes.',
        [
          { text: 'Cancel', style: 'cancel' },
          {
            text: 'Discard',
            style: 'destructive',
            onPress: () => navigation.dispatch(e.data.action),
          },
        ]
      )
    })

    return unsubscribe
  }, [navigation, hasUnsavedChanges])

  return <Form />
}
```

## Project-Specific Patterns

### This Project's Structure

```
app/
├── _layout.tsx              # Root layout with TamaguiProvider
├── (tabs)/
│   ├── _layout.tsx          # Tab navigator
│   ├── index.tsx            # Home tab (/)
│   └── settings.tsx         # Settings tab (/settings)
├── +not-found.tsx           # 404 page
└── +html.tsx                # Custom HTML (web)
```

### Environment Variables for Deep Links

```bash
# .env
EXPO_PUBLIC_APP_SCHEME=mattystack
EXPO_PUBLIC_WEB_URL=https://mattystack.app
```

```typescript
// Access in code
const scheme = process.env.EXPO_PUBLIC_APP_SCHEME
```
