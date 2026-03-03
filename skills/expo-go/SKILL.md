---
name: expo-go
description: Manage Expo Go development workflow for React Native apps. Use when the user mentions Expo Go, starting development server, QR code scanning, testing on physical devices, network issues, running the app on mobile, Expo Router, file-based routing, navigation, tabs, deep linking, or URL schemes.
allowed-tools: Bash, Read, Grep, Glob, WebSearch
---

# Expo Go Development Skill

This skill helps manage Expo Go development workflow for React Native/Expo projects. Expo Go is a sandbox app for rapid mobile development without native build tools.

**Note:** This project uses **bun** as the default package manager. Use `bunx` instead of `npx` and `bun run` instead of `npm run`.

## When to Use This Skill

- Starting development server for Expo Go
- Troubleshooting QR code or network issues
- Testing on physical iOS/Android devices
- Managing development environment
- Understanding Expo Go limitations

## Quick Reference

### Starting Development Server

```bash
# Standard start (LAN mode - recommended for same network)
bunx expo start

# Tunnel mode (when LAN doesn't work)
bunx expo start --tunnel

# Clear cache and start fresh
bunx expo start --clear

# Specify port
bunx expo start --port 8081

# Start with specific connection type
bunx expo start --lan      # LAN mode (default)
bunx expo start --localhost # Local only
bunx expo start --tunnel   # Ngrok tunnel
```

### Development Server Keyboard Shortcuts

When the dev server is running, use these keyboard shortcuts:

| Key | Action |
|-----|--------|
| `a` | Open on Android device/emulator |
| `i` | Open on iOS simulator (Mac only) |
| `w` | Open in web browser |
| `r` | Reload app |
| `m` | Toggle menu |
| `j` | Open debugger |
| `o` | Open project code |
| `c` | Show QR code |
| `?` | Show all commands |

## Connection Modes

### LAN Mode (Default)
- Fastest performance
- Requires same Wi-Fi network
- Device and computer must be on same subnet

### Tunnel Mode
- Works across networks
- Slower reload times
- Requires `@expo/ngrok` (auto-installed)
- Use when: public Wi-Fi, VPN issues, firewall blocking

### Local Mode
- Computer only (localhost)
- For emulator/simulator development

## Expo Router

This project uses **Expo Router** for file-based routing. See [EXPO-ROUTER.md](EXPO-ROUTER.md) for complete documentation.

### Quick Reference

**File Conventions:**
| File | Purpose |
|------|---------|
| `index.tsx` | Default route for directory |
| `[param].tsx` | Dynamic segment |
| `[...rest].tsx` | Catch-all route |
| `_layout.tsx` | Layout wrapper |
| `(group)/` | Route group (no URL segment) |

**Navigation:**
```typescript
import { Link, useRouter } from 'expo-router'

// Declarative
<Link href="/settings">Settings</Link>

// Imperative
const router = useRouter()
router.push('/settings')
router.replace('/home')
router.back()
```

**Reading Params:**
```typescript
import { useLocalSearchParams } from 'expo-router'

const { id } = useLocalSearchParams<{ id: string }>()
```

**This Project's Structure:**
```
app/
├── _layout.tsx          # Root layout with TamaguiProvider
├── (tabs)/
│   ├── _layout.tsx      # Tab navigator
│   ├── index.tsx        # Home tab (/)
│   └── settings.tsx     # Settings (/settings)
└── +not-found.tsx       # 404 page
```

## Deep Linking

### URL Scheme Configuration

This project uses the scheme `mattystack` defined in `app.json`:

```json
{
  "expo": {
    "scheme": "mattystack"
  }
}
```

### Link Formats by Platform

| Platform | Format |
|----------|--------|
| iOS/Android | `mattystack://path` |
| Web | `https://yoursite.com/path` |
| Expo Go | `exp://192.168.x.x:8081/--/path` |

### Testing Deep Links

```bash
# iOS Simulator
xcrun simctl openurl booted "mattystack://settings"

# Android Emulator
adb shell am start -W -a android.intent.action.VIEW -d "mattystack://settings"

# Expo Go (development)
# Use the exp:// URL shown in terminal with /--/path appended
```

## Replit Development

This project is optimized for Replit development with special environment configuration.

### Environment Variables

Use `EXPO_PUBLIC_` prefix for client-accessible variables:

```bash
# .env
EXPO_PUBLIC_API_URL=https://api.example.com
EXPO_PUBLIC_ENV=development
```

Access in code:
```typescript
const apiUrl = process.env.EXPO_PUBLIC_API_URL
```

### Replit-Specific Commands

```bash
# Start with Replit-optimized configuration
bun run expo:dev

# This runs expo with proper host binding for Replit's network
```

### Network Considerations

Replit runs in a container environment. Key considerations:

1. **Use Tunnel Mode**: LAN mode won't work from external devices
   ```bash
   bunx expo start --tunnel
   ```

2. **Webview URL**: Access the web version via Replit's webview URL

3. **Environment Variables**: Set in Replit Secrets, not .env files in production

## EAS Build vs Expo Go

### When to Use Expo Go
- Rapid prototyping
- Testing expo-* packages
- No custom native code needed
- Learning and experimentation

### When to Switch to Development Build
- Need custom native modules
- Using libraries that require native linking
- Push notifications
- In-app purchases
- Specific native configurations

### Creating a Development Build

```bash
# Install EAS CLI
bun add -g eas-cli

# Configure project
eas build:configure

# Create development build
eas build --profile development --platform ios
eas build --profile development --platform android
```

## Troubleshooting Guide

For detailed troubleshooting steps, see [TROUBLESHOOTING.md](TROUBLESHOOTING.md).

### Quick Fixes

**QR Code Won't Scan:**
1. Use native camera app (not third-party scanner)
2. Ensure good lighting
3. Check Expo Go is installed and updated

**"Network request failed" / Can't Connect:**
1. Verify same Wi-Fi network
2. Try tunnel mode: `bunx expo start --tunnel`
3. Check firewall isn't blocking port 8081/19000
4. Disable VPN temporarily

**App Not Updating:**
1. Shake device → "Reload"
2. Press `r` in terminal
3. Restart with `bunx expo start --clear`

## Expo Go Limitations

Expo Go is a **sandbox environment** with pre-bundled native modules. You **cannot**:

- Use custom native code
- Use libraries requiring native linking (e.g., `react-native-firebase`)
- Modify native iOS/Android configuration
- Use push notifications (requires development build)
- Use certain hardware features

### Supported Libraries

All `expo-*` packages work in Expo Go. Check compatibility:
```bash
bunx expo install --check
```

For libraries requiring native code, you must switch to a **development build**.

## Environment Variables

Set public environment variables in your app config:

```json
// app.json
{
  "expo": {
    "extra": {
      "apiUrl": "https://api.example.com"
    }
  }
}
```

Or use `.env` files with `EXPO_PUBLIC_` prefix:
```bash
EXPO_PUBLIC_API_URL=https://api.example.com
```

Access in code:
```javascript
const apiUrl = process.env.EXPO_PUBLIC_API_URL;
```

## Project-Specific Commands

This project uses bun as the package manager. Check `package.json` for:

```bash
# Development server with custom configuration
bun run expo:dev

# Standard expo start
bunx expo start
```

## Best Practices

1. **Keep Expo Go Updated**: Outdated versions cause compatibility issues
2. **Use Same SDK Version**: App SDK must match Expo Go version
3. **Clear Cache When Stuck**: `bunx expo start --clear`
4. **Prefer LAN Over Tunnel**: Better performance when possible
5. **Check Network First**: Most issues are network-related

## SDK Compatibility

Current project SDK: Check `app.json` → `expo.sdkVersion`

Expo Go supports the **latest SDK** and **one previous version**. If your SDK is older, you must either:
- Upgrade SDK: `bunx expo install expo@latest`
- Use a development build

## Common Commands Reference

```bash
# Check Expo CLI version
bunx expo --version

# Login to Expo account
bunx expo login

# Check current user
bunx expo whoami

# Install compatible package versions
bunx expo install [package-name]

# Check for dependency issues
bunx expo doctor

# View project config
bunx expo config

# Clear all caches
bunx expo start --clear
watchman watch-del-all  # If using watchman
rm -rf node_modules/.cache
```

## Device Setup

### Android
1. Install Expo Go from Google Play Store
2. Enable Developer Options (tap Build Number 7 times)
3. Enable USB Debugging (optional, for USB connection)
4. Scan QR code with Expo Go app

### iOS
1. Install Expo Go from App Store
2. Scan QR code with Camera app
3. Tap the notification to open in Expo Go

## Additional Resources

- [EXPO-ROUTER.md](EXPO-ROUTER.md) - Complete Expo Router reference
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) - Detailed troubleshooting guide
- [COMMANDS.md](COMMANDS.md) - Complete command reference
