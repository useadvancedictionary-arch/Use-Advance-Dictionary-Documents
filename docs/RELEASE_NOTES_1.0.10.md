# USE Advanced Dictionary — Release 1.0.10 (build 11)

**Version:** 1.0.10+11  
**Date:** 22 May 2026

## Simple version guide

| Where | What you see |
|-------|----------------|
| **Play Store now (live)** | **10 (1.0.9)** — version code **10**, name **1.0.9** |
| **Upload this build** | **11 (1.0.10)** — version code **11**, name **1.0.10** |

The first number in Play (**10** → **11**) must increase each upload. The name (**1.0.9** → **1.0.10**) is what users read as the app version.

## What's new

- **Show external words** (Settings → Dictionary) — **off by default**. When on, missing words can use **online English-only** definitions; Search shows **Look up "…" online**.
- **Keyboard fixes** on Language translator, Grammar Section, and Practice Quiz (no black gap above keyboard).
- **Multi-device sync** dialog scrolls correctly on small screens.
- **Android edge-to-edge** alignment (addresses Play Console recommendations).

## Play Store upload

1. Production → **Create new release**
2. Upload: `build/app/outputs/bundle/release/app-release.aab`
3. Confirm: **11 (1.0.10)**
4. Release notes (short):

```
What's new in 1.0.10:
• Optional online English lookup for words not in the offline dictionary (Settings → Dictionary).
• Search: "Look up online" when no local match (when setting is on).
• Keyboard layout fixes on Translator, Grammar, and Practice.
• Sync settings dialog scroll fix.
```

## Windows

- EXE folder: `build\windows\x64\runner\Release\` (ship whole folder, not only .exe)
- APK (side load): `build\app\outputs\flutter-apk\app-release.apk`
