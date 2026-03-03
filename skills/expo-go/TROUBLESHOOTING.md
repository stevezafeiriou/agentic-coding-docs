# Expo Go Troubleshooting Guide

Comprehensive troubleshooting for common Expo Go issues.

## Connection Issues

### "Network request failed" / Cannot Connect to Dev Server

**Symptoms:**
- QR code scans but app won't load
- "Something went wrong" error
- Timeout errors in Expo Go

**Solutions (try in order):**

1. **Verify Same Network**
   ```bash
   # Check your computer's IP
   # macOS/Linux:
   ifconfig | grep "inet "
   # Windows:
   ipconfig
   ```
   Ensure your phone's Wi-Fi shows the same network name.

2. **Check Wi-Fi Band**
   - 2.4GHz and 5GHz networks may be isolated
   - Connect both devices to same band
   - Try the 2.4GHz network (wider range)

3. **Use Tunnel Mode**
   ```bash
   bunx expo start --tunnel
   ```
   This bypasses local network entirely.

4. **Firewall/Antivirus**
   - Temporarily disable firewall
   - Allow Node.js through firewall
   - Whitelist ports: 8081, 19000, 19001

5. **Disable VPN**
   VPNs route traffic differently and break LAN connections.

6. **Router AP Isolation**
   - Log into router admin panel
   - Disable "AP Isolation" or "Client Isolation"
   - Common on guest networks

### Connection Works But Very Slow

**Solutions:**
1. Switch from Tunnel to LAN mode
2. Clear Metro cache: `bunx expo start --clear`
3. Close unnecessary apps on device
4. Restart development server

## QR Code Issues

### QR Code Won't Scan

**Checklist:**
- [ ] Using native Camera app (iOS) or Expo Go scanner (Android)
- [ ] Good lighting on QR code
- [ ] Phone camera lens is clean
- [ ] Not using third-party QR scanner app

**iOS Specific:**
1. Open Camera app
2. Point at QR code
3. Tap the notification banner that appears
4. If no banner: open Expo Go → "Enter URL manually"

**Android Specific:**
1. Open Expo Go app directly
2. Use built-in "Scan QR Code" button
3. Grant camera permissions if prompted
4. Grant location permissions (required for LAN)

### "No usable data found" When Scanning

This happens when:
- Using wrong QR scanning app
- Expo Go not installed
- Deep link format not recognized

**Fix:**
1. Install/update Expo Go from app store
2. Use Expo Go's built-in scanner (Android)
3. Use native Camera (iOS)

## App Loading Issues

### App Stuck on Splash Screen

```bash
# Clear all caches and restart
bunx expo start --clear

# If still stuck, clear more aggressively:
rm -rf node_modules/.cache
rm -rf .expo
bun run start -- --clear
```

### "Invariant Violation" Errors

Usually means a native module mismatch:

1. **Check SDK Version Match**
   ```bash
   # Your project SDK (in app.json)
   cat app.json | grep sdkVersion

   # Expo Go version should match
   ```

2. **Update Expo Go App**
   - Delete and reinstall from app store
   - Or update to latest version

3. **Check for Incompatible Libraries**
   ```bash
   bunx expo doctor
   ```

### JavaScript Bundle Failed to Load

```bash
# Reset Metro bundler cache
bunx expo start --clear

# More aggressive reset
rm -rf node_modules
rm -rf .expo
bun install
bunx expo start --clear
```

## SDK Version Mismatch

### "This project uses SDK XX but Expo Go supports SDK YY"

**Options:**

1. **Upgrade Your Project (Recommended)**
   ```bash
   # Upgrade to latest SDK
   bunx expo install expo@latest

   # Then fix any dependency issues
   bunx expo doctor --fix-dependencies
   ```

2. **Update Expo Go**
   - Get latest version from app store

3. **Use Development Build**
   - If you need specific SDK version
   - See Expo docs on development builds

## Hot Reload Not Working

### Changes Not Appearing

1. **Manual Reload**
   - Shake device → "Reload"
   - Press `r` in terminal
   - Ctrl+M (Android emulator) or Cmd+D (iOS simulator)

2. **Check Fast Refresh**
   - Shake device → "Enable Fast Refresh"

3. **Clear Cache**
   ```bash
   bunx expo start --clear
   ```

4. **Check for Syntax Errors**
   - Look at terminal for red error messages
   - Fix errors before expecting reload

### "Unable to resolve module" Errors

```bash
# Install missing dependencies
bunx expo install

# Clear Metro cache
bunx expo start --clear

# Full reset
rm -rf node_modules
bun install
bunx expo start --clear
```

## Device-Specific Issues

### Android: "Something went wrong"

1. **Enable Location Permission**
   - Settings → Apps → Expo Go → Permissions
   - Enable Location (required for LAN discovery)

2. **Clear Expo Go Data**
   - Settings → Apps → Expo Go → Storage → Clear Data

3. **Reinstall Expo Go**

### iOS: App Closes Immediately

1. **Trust Developer Certificate** (if using dev builds)
   - Settings → General → Device Management

2. **Restart Device**

3. **Reinstall Expo Go**

### iOS Simulator: "Unable to boot"

```bash
# Reset simulator
xcrun simctl shutdown all
xcrun simctl erase all
```

## Performance Issues

### Slow Development Experience

1. **Use LAN Instead of Tunnel**
   ```bash
   bunx expo start --lan
   ```

2. **Reduce Bundle Size**
   - Remove unused imports
   - Use dynamic imports for large libraries

3. **Disable React Compiler** (temporarily)
   ```javascript
   // babel.config.js
   module.exports = {
     presets: ['babel-preset-expo'],
     // Comment out react compiler plugin temporarily
   };
   ```

4. **Use Physical Device**
   - Simulators/emulators are slower
   - Physical devices give better performance testing

## Environment Variable Issues

### Variables Not Loading

1. **Use Correct Prefix**
   ```bash
   # Must start with EXPO_PUBLIC_
   EXPO_PUBLIC_API_URL=https://api.example.com  # ✓
   API_URL=https://api.example.com              # ✗ won't work
   ```

2. **Restart Dev Server**
   ```bash
   # Env vars are loaded at startup
   bunx expo start --clear
   ```

3. **Check .env File Location**
   - Must be in project root
   - Named `.env` or `.env.local`

## Getting More Help

### Useful Diagnostic Commands

```bash
# Check Expo environment
bunx expo doctor

# View full project config
bunx expo config --full

# Check for outdated packages
bunx expo install --check

# View Expo CLI version
bunx expo --version

# View React Native version
bunx react-native --version
```

### Log Collection

```bash
# Enable verbose logging
DEBUG=expo:* bunx expo start

# View device logs (Android)
adb logcat | grep -i expo

# View device logs (iOS Simulator)
xcrun simctl spawn booted log stream --predicate 'process == "Expo Go"'
```

### Where to Get Help

1. **Expo Discord**: https://chat.expo.dev
2. **Expo Forums**: https://forums.expo.dev
3. **GitHub Issues**: https://github.com/expo/expo/issues
4. **Stack Overflow**: Tag `expo` and `react-native`
