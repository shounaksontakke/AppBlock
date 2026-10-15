# FocusLock

FocusLock is an offline Android app for controlling when distracting apps and
websites can be used. It blocks apps and sites on schedules, applies daily usage
limits, runs focus sessions, and shows screen-time statistics.

It has no account, no server and no `INTERNET` permission. Nothing it learns
about your phone use can leave the device.

FocusLock is a self-control tool, not a lock. See
[Android limitations](#android-limitations) for what it cannot do.

## Contents

- [Install](#install)
- [Initial setup](#initial-setup)
- [Example](#example)
- [Features](#features)
- [Privacy](#privacy)
- [Android limitations](#android-limitations)
- [Building and testing](#building-and-testing)
- [Project layout](#project-layout)
- [Audit and test results](#audit-and-test-results)

## Install

Requirements: JDK 17 or newer, and the Android SDK with platform `android-37.0`
and build-tools 36. Point `local.properties` at the SDK:

```properties
sdk.dir=/path/to/Android/Sdk
```

Build the APK:

```bash
./gradlew assembleRelease
```

The signed APK is written to `app/build/outputs/apk/release/app-release.apk`.
Release signing reads `keystore.properties` in the project root (see
`keystore.properties.example`). Without that file the build still succeeds but
produces `app-release-unsigned.apk`, which Android will not install.

Install the APK:

```bash
adb install app/build/outputs/apk/release/app-release.apk
```

or copy the file to the phone and open it. Then open FocusLock.

FocusLock runs on Android 8.0 (API 26) and later.

## Initial setup

The first launch walks through five screens. Nothing is blocked unless you
choose it.

1. **Select applications.** Pick the apps you find distracting.
2. **Create a schedule.** The suggestion is every day, 9:00 PM to 9:00 AM.
3. **Enable Usage Access.** Needed for screen-time statistics and daily limits.
4. **Enable Accessibility.** Needed for blocking. In Accessibility settings,
   turn on **FocusLock app blocking**.
5. **Create a PIN** (optional). Offered on the last screen and under
   Settings, Security.

If you installed the APK from a file on Android 13 or later, Android may show
"Restricted setting" when you try to enable the accessibility service. Open
FocusLock's App info, tap the menu in the top corner, choose **Allow restricted
settings**, then try again. Apps installed with `adb install` are not affected.

Website blocking is separate and optional: turn on **FocusLock website
blocking** in Accessibility settings (Settings, Websites in the app explains
it).

Settings, Permissions shows the state of every permission and whether blocking
is working.

## Example

```text
Instagram
21:00 → 09:00
Every day
Strict Mode ON
```

1. Apps tab, or Settings, Schedules, New schedule.
2. Name it, leave the days on "Every day", set Start 9:00 PM and End 9:00 AM.
   The editor marks the end as "The next day".
3. Blocked apps: choose Instagram. Blocked websites: add `instagram.com`.
4. Turn on Strict Mode and save.

From 21:00 to 09:00, opening Instagram shows the block screen with the reason
and the time it becomes available. Because the schedule is strict, the rules
cannot be edited while it runs.

## Features

**App blocking**

| Mechanism | What it does |
|---|---|
| Always block | The app is blocked at all times. |
| Schedule | Blocked on chosen days between chosen times. Overnight windows, several windows per day and overlapping schedules are supported. |
| Daily usage limit | Blocked for the rest of the day once the limit is used. |
| Opens per day | Blocked after it has been opened the set number of times. |
| Cooldown | After a set amount of use, locked for a set time. |
| Category | Every app in a category (Social, Games, and so on) is blocked. |

**Website blocking.** Domains, with subdomains included, in supported browsers.
Lists can be pasted or imported from a text file, and there are built-in
categories.

**Focus sessions.** 25, 45, 60 or 90 minutes or a custom length, with a
configurable emergency exit: end any time, wait then end, PIN required, or no
early exit.

**Breaks.** Short pauses taken from the block screen, limited per day, and only
when the active rule and Strict Mode allow them. Breaks never lift usage limits.

**Strict Mode.** Locks schedule editing, blocked-app and website changes, limit
changes, focus cancellation, PIN changes and the blocking switch. It is active
during a strict schedule, a strict focus session, or a manual timed lock.
Changes that make blocking stricter are still allowed.

**PIN.** 4-digit, 6-digit or custom length. Optional biometric unlock.
Accountability Mode lets a trusted person set the PIN.

**Always allowed apps.** Apps you choose are never blocked. Phone, Android
Settings, the launcher, the keyboard, the permission controller and the package
installer can never be blocked by any rule.

**Statistics.** Screen time, pickups, per-app usage, daily and weekly views,
blocked attempts, focus time and limits reached. Export as CSV or JSON.

**Policy order.** One engine decides every block, in this order: master switch,
system and user allowlist, break, Strict Mode, focus session, schedules, usage
limits, always-on app and category rules.

## Privacy

### What stays on the device

Everything. FocusLock declares no `INTERNET` permission, and Android does not
let an app without it open a network connection. There is no account, backend,
sync, analytics, advertising, crash reporting or tracking code.

### Permissions

| Permission | Why |
|---|---|
| `PACKAGE_USAGE_STATS` | Screen-time statistics and daily limits. Granted in Usage access settings. |
| `POST_NOTIFICATIONS` | Reminders, limit notices and the focus timer. Each kind can be switched off. |
| `RECEIVE_BOOT_COMPLETED` | Re-register the reminder alarm after a restart. |
| `SCHEDULE_EXACT_ALARM` | Optional. On-time reminders. Blocking does not depend on it. |
| `USE_BIOMETRIC`, `USE_FINGERPRINT` | Optional biometric unlock. `USE_FINGERPRINT` is added by the AndroidX biometric library for older Android versions. |
| `com.focuslock.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION` | Generated by AndroidX. A signature-level permission that keeps FocusLock's own internal broadcasts private. |

Not requested: `INTERNET`, `QUERY_ALL_PACKAGES`, `SYSTEM_ALERT_WINDOW`, any
foreground-service permission, battery-optimisation exemption, device admin,
location, contacts, storage, camera or microphone.

### What the two accessibility services can see

**FocusLock app blocking** is declared with `canRetrieveWindowContent="false"`
and listens only for window-state changes. From each event it reads the event
type and the package name. It cannot read text, passwords, messages or anything
on screen, and it does not record keystrokes or interactions.

**FocusLock website blocking** is optional. It needs to read one piece of text,
the browser's address bar, so it is a separate service that you enable
separately. It receives events only from supported browsers and looks up a
single view by ID. The address is reduced to a hostname in memory and compared
with your block list. Hosts that are not blocked are discarded immediately.
When a blocked site is opened, the attempt is counted against the rule that
matched (for example `instagram.com`), never the page you visited. It does not
read page content or form fields.

### What is stored

- Your rules and settings.
- One line per app per day: usage time, opens, blocked attempts, notification
  count. Daily totals. Focus sessions.
- A PIN verifier: PBKDF2-HMAC-SHA256 with a random salt, then an HMAC under a
  non-exportable key in Android Keystore. The PIN itself is never stored or
  logged.

No stream of events, timestamps of individual launches, window titles or URLs
is kept. Storage is app-private and excluded from Android cloud backup and
device-to-device transfer.

You can export or delete usage history, reset the rules, or delete all data
under Settings, History and export.

### Usage Access

Screen time comes from Android's own usage statistics. FocusLock reads them
through Usage Access, folds them into per-app totals in memory, and stores only
the totals.

### Dependencies

The release build has 149 library modules. All but four are AndroidX (Compose,
Material 3, Navigation, Lifecycle, Room, DataStore, Biometric and what they
pull in), the Kotlin standard library, or kotlinx coroutines and serialization.
The other four come in through AndroidX: Okio (file I/O used by DataStore),
Guava's `ListenableFuture` interface, and JSpecify annotations. All are Apache
2.0. None is an analytics, advertising, crash-reporting or HTTP library. Compose bundles EmojiCompat, whose default start-up asks a system font
provider for a downloadable emoji font; FocusLock removes that initialiser.

## Android limitations

FocusLock cannot be made impossible to bypass, and does not claim to be. The
owner of an Android device controls the operating system.

**Accessibility can be turned off.** Anyone with the phone can disable the
accessibility service in Android Settings, and blocking stops. FocusLock never
blocks Android Settings or permission screens. It posts a notification when the
service is switched off, and Settings, Permissions shows the state.

**Uninstalling and clearing data.** FocusLock can be uninstalled, and its data
can be cleared from Android Settings. Either removes all rules and the PIN.
There is no uninstall protection: FocusLock does not use Device Admin.

**Safe Mode.** Starting the phone in Safe Mode disables third-party apps,
including FocusLock.

**Device owner and profiles.** FocusLock is an ordinary app, not a device
owner. It does not control other user profiles, a work profile, guest mode or
private space. Apps opened there are not blocked.

**Forgotten PIN.** There is no recovery and no hidden credential. Clearing
FocusLock's data resets everything.

**Clock changes.** Schedules follow the device clock, so changing the time or
time zone moves them. Focus sessions, breaks, cooldowns, timed Strict Mode
locks and PIN lockouts are measured against a clock the user cannot set, and
are not shortened by a clock change (until the next restart, after which the
wall clock is used).

**Restricted settings.** Sideloaded apps on Android 13 and later need "Allow
restricted settings" before an accessibility service can be enabled.

**OEM battery management.** Some manufacturers stop background components.
If blocking stops by itself, allow FocusLock to run in the background.
Settings, Permissions, "Help for your phone's battery settings" has steps for
Pixel, Samsung, Xiaomi, OnePlus, Oppo, Realme, Vivo, Motorola and Huawei.
Menu names differ between models, and behaviour is not the same on every phone.

**A short flash.** Blocking reacts after Android reports that an app has come
to the front, so a blocked app can be visible for a fraction of a second before
it is covered.

**Split screen and picture-in-picture.** The app-blocking service cannot see
window layout, so an app shown in a second window is detected only when it
takes focus.

**Website blocking covers supported browsers only.** It works by reading the
address bar in Chrome and Chromium-based browsers, Firefox, Samsung Internet,
Edge, Brave, Opera, Vivaldi and DuckDuckGo. It does not cover web pages opened
inside other apps, or browsers that are not listed. Browsers change their
internals, so support for one can break after a browser update. While a browser
shows one of its own dialogs (a permission prompt, for example), Android hides
the page behind it from accessibility services, so a blocked site is caught
only once the dialog is dismissed. An option
blocks unsupported browsers entirely while website rules are active.

**No VPN or DNS filtering.** A local VPN could block domains in every app, but
a VPN app must forward DNS lookups, which requires the `INTERNET` permission.
That conflicts with FocusLock's central privacy guarantee, so it is not
included. VPN-based filtering is also bypassed by Private DNS and by browsers
with DNS over HTTPS, and it occupies the phone's single VPN slot. FocusLock
does not intercept or decrypt traffic and installs no certificates.

**Statistics.** Android keeps detailed usage events for about a week, so
history starts when Usage Access is granted and builds up from there. The
notification count relies on usage events that Android reports; where it does
not, the app shows "Not available". Pickups count device unlocks.

**Reminders.** Without the optional "Alarms and reminders" permission,
reminders may arrive a few minutes late. Blocking is not affected.

## Building and testing

```bash
./gradlew test                      # JVM unit tests
./gradlew lint                      # Android lint
./gradlew assembleDebug             # debug APK
./gradlew assembleRelease           # signed, minified release APK
./gradlew connectedDebugAndroidTest # Compose UI and storage tests, on a device
scripts/e2e_emulator.sh             # end-to-end blocking checks over adb
```

Check the permissions of the built APK:

```bash
$ANDROID_HOME/cmdline-tools/latest/bin/apkanalyzer manifest permissions \
    app/build/outputs/apk/release/app-release.apk
```

`scripts/e2e_emulator.sh` installs the debug build on a running emulator, turns
on both accessibility services and usage access over adb, writes rules into the
app's storage, launches apps and checks which activity Android reports as
resumed. It covers always-block, the allowlist, schedules, an app already open
when its schedule starts, daily limits, focus sessions, breaks, website
blocking in Chrome, and a reboot.

`connectedDebugAndroidTest` also renders every screen in light, dark, tablet,
small-phone and large-text variants and saves the images under
`app/build/outputs/connected_android_test_additional_output/`.

**Emulator on Fedora.** With SELinux enforcing, the emulator's default software
renderer is denied `execheap` and crashes. Start the emulator with
`-gpu swangle_indirect`. If a crashed first boot leaves the virtual device in a
boot loop, start it once with `-wipe-data`.

`./gradlew test` runs the unit tests for the debug build type, and
`./gradlew lint` checks the debug variant. `assembleRelease` runs lint's fatal
checks on the release variant.

## Project layout

```text
app/src/main/java/com/focuslock/
├── MainActivity.kt, FocusLockApp.kt
├── di/              AppContainer: the object graph, built by hand
├── core/            clock abstraction, text formatting
├── model/           rules, runtime state, deadlines
├── blocker/         PolicyEngine, StrictGuard, BlockEngine, repositories
├── scheduler/       ScheduleEngine, alarms, boot restore, housekeeping
├── usage/           usage-event aggregation and history
├── website/         domain matching, browser registry, website blocker
├── accessibility/   the two accessibility services
├── security/        PIN, Keystore, biometrics
├── focus/           focus sessions, breaks, timed Strict Mode locks
├── apps/            installed-app discovery and categories
├── notifications/   notification channels and messages
├── permissions/     permission state and manufacturer advice
├── data/            DataStore files, Room database, export
└── ui/              Compose screens and components
```

`PolicyEngine` and `ScheduleEngine` are pure Kotlin with no Android imports:
every decision is a function of its arguments. Rules are one JSON document in
DataStore; history is in Room. Every change to the rules is diffed against the
stored value and passed through `StrictGuard` in the repository, so no screen
can weaken the rules without authorisation.

## Audit and test results

Recorded on 2026-10-08 against the release APK and an Android 17 (API 37)
emulator. Nothing below was run on a physical phone, on another Android
version, or on a manufacturer's modified Android.

### Release APK

| Check | Result |
|---|---|
| Build | `./gradlew assembleRelease` succeeds; R8 minification and resource shrinking on |
| File | `app/build/outputs/apk/release/app-release.apk`, about 2.6 MB |
| Signature | `apksigner verify` passes, APK Signature Scheme v2 |
| Debuggable | `false` |
| SDK levels | min 26, target 37 |
| Backup | `allowBackup="false"`, with data-extraction rules that exclude everything |
| Cleartext traffic | `usesCleartextTraffic="false"` (and no network permission at all) |

`apkanalyzer manifest permissions app-release.apk` prints:

```text
android.permission.PACKAGE_USAGE_STATS
android.permission.SCHEDULE_EXACT_ALARM
android.permission.USE_BIOMETRIC
android.permission.USE_FINGERPRINT
android.permission.POST_NOTIFICATIONS
android.permission.RECEIVE_BOOT_COMPLETED
com.focuslock.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION
```

`android.permission.INTERNET` is absent.

### Components

| Component | Exported | Guard |
|---|---|---|
| `MainActivity` | yes | Launcher entry. Reads one extra, a route name checked against a fixed list. |
| `BlockedActivity` | no | Only FocusLock's services can start it. The host it is given is re-validated. |
| `FocusAccessibilityService` | yes | `BIND_ACCESSIBILITY_SERVICE`: only the system can bind. |
| `BrowserAccessibilityService` | yes | `BIND_ACCESSIBILITY_SERVICE`: only the system can bind. |
| `AlarmReceiver` | no | Reached only through FocusLock's own immutable `PendingIntent`. |
| `BootReceiver` | no | System broadcasts only; the action is checked. |
| `ProfileInstallReceiver` (AndroidX) | yes | `DUMP` permission: shell and system only. |
| `InitializationProvider` (AndroidX) | no | EmojiCompat's initialiser removed. |

There are no deep links, no `WebView`, no content provider of FocusLock's own,
and no debug-only component in the release manifest.

With both services enabled, `dumpsys accessibility` on the emulator reports
`capabilities=0` for "FocusLock app blocking" and `capabilities=1` (retrieve
window content) for "FocusLock website blocking". The operating system itself
confirms the app-blocking service cannot read the screen.

### Code

- No logging calls in `app/src/main`. R8 also strips `Log.v/d/i` from release.
- No use of sockets, `HttpURLConnection`, `WebView`, dynamic code loading or
  process execution in `app/src/main`.
- PIN: PBKDF2-HMAC-SHA256, 310,000 iterations, 16-byte random salt, bound by an
  HMAC key held in Android Keystore. Verified on the emulator with the real
  Keystore. The PIN entry screens set `FLAG_SECURE`; a screenshot of them comes
  out black.
- Lint: 0 issues on the debug variant; fatal checks pass on release.

### Tests

| Suite | Result |
|---|---|
| JVM unit tests (`./gradlew test`) | 176 passed |
| Instrumented tests (`connectedDebugAndroidTest`) | 90 passed |
| End-to-end script (`scripts/e2e_emulator.sh`) | 32 passed, 0 failed |

The unit tests cover the `21:00 → 09:00` boundaries (20:59 allowed, 21:00,
23:59, 00:00 and 08:59 blocked, 09:00 allowed), `09:00 → 17:00` and
`23:30 → 00:30`, day sets, several windows, overlapping and disabled
schedules, daylight-saving changes in New York and Berlin, time-zone changes,
the daily limit at 29, 30 and 31 minutes, the policy order, Strict Mode and PIN
rules, refusal of protected changes at the storage layer, and restoring rules
and running sessions after a restart.

The instrumented tests cover onboarding, the dashboard, app selection, schedule
creation, PIN setup, Strict Mode, the block screen, usage statistics, settings
and permission health, plus Keystore and Room on the device.

The end-to-end script checks, on a real install: always-block, the allowlist,
Android Settings never being blocked, the master switch, a schedule that
covers the present and then ends, an app already open when its schedule
starts, a daily limit, a focus session, a break, Strict Mode voiding a break,
a blocked site in Chrome, and blocking after a reboot without FocusLock being
opened.

### Checked by hand on the emulator

On the release build: onboarding creating only the schedule that was chosen,
blocking an app from the Apps tab, a break taken from the block screen, and a
focus session with its ongoing notification and the "wait, then end" exit.

On the debug build, where stored state can be inspected: setting a PIN through
the UI stores only a salted verifier; unblocking an app then needs the PIN, a
wrong PIN changes nothing and is counted, the right PIN applies the change;
with Strict Mode locked and PIN override off, unblocking and switching
blocking off are both refused.

### Not verified

- Any physical phone, any manufacturer's battery manager, or any Android
  version other than 17.
- Biometric unlock: the emulator has no biometrics enrolled.
- CSV and JSON export through the system file picker. The formatting is unit
  tested; the picker flow was not driven.
- Browsers other than Chrome. The address-bar IDs listed for them were not
  exercised.
- TalkBack. Content descriptions, roles and headings are set and checked in UI
  tests, but no screen reader session was run.
