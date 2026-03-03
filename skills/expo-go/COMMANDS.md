# Expo Go Command Reference

Complete reference for Expo CLI commands relevant to Expo Go development.

**Note:** This project uses **bun** as the default package manager. Use `bunx` instead of `npx`.

## Development Server Commands

### bunx expo start

Start the development server for Expo Go.

```bash
# Basic start
bunx expo start

# Shorthand alias
bunx expo
```

**Options:**

| Flag | Description |
|------|-------------|
| `--clear`, `-c` | Clear Metro bundler cache |
| `--port <port>` | Specify port (default: 8081) |
| `--lan` | Use LAN connection mode |
| `--localhost` | Use localhost connection mode |
| `--tunnel` | Use tunnel connection mode (ngrok) |
| `--offline` | Skip network connectivity check |
| `--https` | Enable HTTPS (requires setup) |
| `--dev` | Development mode (default: true) |
| `--no-dev` | Production mode bundle |
| `--minify` | Minify JavaScript bundle |
| `--scheme <scheme>` | Custom URL scheme |
| `--go` | Force Expo Go compatibility mode |

**Examples:**

```bash
# Start with tunnel for remote testing
bunx expo start --tunnel

# Start on custom port
bunx expo start --port 3000

# Start fresh (clear cache)
bunx expo start --clear

# Start with production bundle
bunx expo start --no-dev --minify
```

## Interactive Terminal Commands

When dev server is running, press these keys:

| Key | Action |
|-----|--------|
| `a` | Open Android (device or emulator) |
| `i` | Open iOS Simulator (Mac only) |
| `w` | Open web browser |
| `r` | Reload app |
| `m` | Toggle dev menu |
| `j` | Open Chrome DevTools debugger |
| `o` | Open project in editor |
| `c` | Show QR code in terminal |
| `d` | Toggle development mode |
| `shift+a` | Select Android device/emulator |
| `shift+i` | Select iOS Simulator |
| `?` | Show all available commands |
| `ctrl+c` | Stop server |

## Package Management

### bunx expo install

Install packages with correct versions for your SDK.

```bash
# Install specific package
bunx expo install expo-camera

# Install multiple packages
bunx expo install expo-camera expo-location expo-notifications

# Check for version mismatches
bunx expo install --check

# Fix version mismatches
bunx expo install --fix

# Force specific package manager (bun is default for this project)
bunx expo install expo-camera --bun
bunx expo install expo-camera --npm
bunx expo install expo-camera --yarn
bunx expo install expo-camera --pnpm
```

## Project Information

### bunx expo config

View merged app configuration.

```bash
# Show full config
bunx expo config --full

# Output as JSON
bunx expo config --json

# Show specific platform
bunx expo config --platform ios
bunx expo config --platform android
```

### bunx expo doctor

Check project for common issues.

```bash
# Run diagnostics
bunx expo doctor

# Auto-fix dependency issues
bunx expo doctor --fix-dependencies
```

## Account Management

### bunx expo login

Log in to your Expo account.

```bash
bunx expo login

# Login with username/password directly
bunx expo login -u username -p password
```

### bunx expo logout

Log out of Expo account.

```bash
bunx expo logout
```

### bunx expo whoami

Show currently logged-in user.

```bash
bunx expo whoami
```

### bunx expo register

Create a new Expo account.

```bash
bunx expo register
```

## Project Customization

### bunx expo customize

Generate config files for customization.

```bash
# Interactive selection
bunx expo customize

# Generate specific files
bunx expo customize metro.config.js
bunx expo customize babel.config.js
bunx expo customize tsconfig.json
```

## Cache Management

### Clearing Caches

```bash
# Clear Metro bundler cache (most common)
bunx expo start --clear

# Clear Expo cache
rm -rf .expo

# Clear Metro cache
rm -rf node_modules/.cache/metro

# Clear watchman cache (if using)
watchman watch-del-all

# Nuclear option - clear everything
rm -rf node_modules .expo
bun install
bunx expo start --clear
```

## Environment Variables

### Setting Environment Variables

```bash
# In .env file (must use EXPO_PUBLIC_ prefix)
EXPO_PUBLIC_API_URL=https://api.example.com
EXPO_PUBLIC_DEBUG=true

# Pass inline
EXPO_PUBLIC_API_URL=https://staging.api.com bunx expo start
```

### Accessing in Code

```javascript
// Direct access
const apiUrl = process.env.EXPO_PUBLIC_API_URL;

// Via Constants (legacy method)
import Constants from 'expo-constants';
const apiUrl = Constants.expoConfig?.extra?.apiUrl;
```

## Debugging Commands

### Enable Verbose Logging

```bash
# All Expo debug logs
DEBUG=expo:* bunx expo start

# Specific subsystems
DEBUG=expo:metro bunx expo start
DEBUG=expo:start bunx expo start
```

### Device Logs

```bash
# Android device logs
adb logcat *:S ReactNative:V ReactNativeJS:V

# iOS Simulator logs
xcrun simctl spawn booted log stream --predicate 'subsystem == "com.facebook.react"'
```

## Project URLs

When the dev server starts, it provides URLs in different formats:

```
Metro waiting on:
  › LAN:    exp://192.168.1.100:8081
  › Tunnel: exp://xxxxx.expo.dev
  › Local:  exp://localhost:8081
```

### URL Schemes

| Scheme | Use Case |
|--------|----------|
| `exp://` | Expo Go development |
| `exps://` | Expo Go with HTTPS |
| Custom scheme | Development builds |

## Project Scripts (This Project)

This project uses bun as the package manager:

```bash
# Start with project-specific configuration
bun run expo:dev

# Alternative: standard expo start
bunx expo start

# Check types
bun run check:types

# Lint code
bun run lint
bun run lint:fix

# Format code
bun run format
bun run check:format
```

## Expo Router Commands

### Generate Route Types

```bash
# Generate typed routes (after adding expo.experiments.typedRoutes)
bunx expo customize tsconfig.json
```

### Deep Link Testing

```bash
# Test deep links on iOS Simulator
xcrun simctl openurl booted "myapp://path/to/screen"

# Test deep links on Android Emulator
adb shell am start -W -a android.intent.action.VIEW -d "myapp://path/to/screen"

# Open specific route in Expo Go (development)
# Append path to your exp:// URL: exp://192.168.x.x:8081/--/path/to/screen
```

### Route Debugging

```bash
# Log all routes in the app
bunx expo routes

# Generate static routes manifest
bunx expo export --platform web --output-dir dist
cat dist/_expo/routes.json
```

### Common Route Patterns

```bash
# Create a new route file
# app/settings.tsx → /settings
# app/users/[id].tsx → /users/:id
# app/(tabs)/index.tsx → / (in tab group)

# Create a layout file
# app/_layout.tsx → Root layout
# app/(tabs)/_layout.tsx → Tab layout
```

## Common Workflows

### Fresh Start Workflow

```bash
# 1. Clear everything
rm -rf node_modules .expo
bun install

# 2. Start fresh
bunx expo start --clear
```

### Testing on Device Workflow

```bash
# 1. Start server
bunx expo start

# 2. If LAN fails, use tunnel
bunx expo start --tunnel

# 3. Scan QR code with device
```

### Debugging Workflow

```bash
# 1. Start with debugging
bunx expo start

# 2. Press 'j' to open debugger
# 3. Or shake device → "Debug Remote JS"
```

### Update Dependencies Workflow

```bash
# 1. Check what needs updating
bunx expo install --check

# 2. Fix versions
bunx expo install --fix

# 3. Run doctor
bunx expo doctor

# 4. Clear cache and restart
bunx expo start --clear
```
