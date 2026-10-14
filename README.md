# SnapTalk 💬

A real-time chat app built with **Flutter**, **Firebase Authentication** (Email/Password), and **Cloud Firestore** — with a **Global Chat** room plus **searchable 1-to-1 direct messages**.

---

## Features

- **Sign Up / Log In** with name, email, and password (Firebase Auth)
- **Global Chat** — one shared room everyone can see and post in
- **Find people by name or email** and start a private 1-to-1 chat instantly
- **Real-time messaging** — messages sync instantly across devices (Cloud Firestore + Streams)
- **Recent DMs list** on the home screen, most recently active first
- Auto-navigation: logged-in users go straight to the home screen, logged-out users see the login screen
- Simple, readable codebase — good for explaining in a viva/exam

---

## Project Structure

```
lib/
 ├── main.dart                   # App entry point + AuthGate (decides login vs home screen)
 ├── models/
 │    ├── app_user.dart          # Public user profile (uid, name, email) — used for search
 │    ├── chat.dart              # Chat metadata (global room or 1-to-1 DM)
 │    └── message.dart           # Message data model
 ├── services/
 │    ├── auth_service.dart      # Sign up / Log in / Log out + saves user profile on signup
 │    └── chat_service.dart      # Global chat, DM creation, user search, send/stream messages
 ├── screens/
 │    ├── login_screen.dart      # Login/Signup UI (Name field shown on Sign Up)
 │    ├── home_screen.dart       # Global Chat tile + recent DMs + search icon
 │    ├── search_screen.dart     # Search accounts by name/email → tap to open a DM
 │    └── chat_screen.dart       # Generic chat UI (used for both global & DM chats)
 └── widgets/
      └── message_bubble.dart    # Single chat bubble UI
```

This project is already configured with a Firebase project (`snaptalk-536c7`) and
package name `com.nishat.snaptalk`. The `android/app/google-services.json` file
is already included.

---

## Setup Instructions (Before Running)

### 1. Install Flutter
Make sure Flutter is installed: https://docs.flutter.dev/get-started/install

Check with:
```bash
flutter doctor
```

### 2. Enable Firebase services (IMPORTANT — do this first)
Go to the [Firebase Console](https://console.firebase.google.com/) → open project **snaptalk-536c7**:

1. **Authentication** → Get Started → Sign-in method → enable **Email/Password**
2. **Firestore Database** → Create database → Start in **test mode** (for quick exam use — see note below)
3. Under **Project settings → Authorized domains**, `localhost` is included by
   default — this is what lets `flutter run -d chrome` work for Auth during
   local development.
4. **Publish the updated Firestore rules** (see `firestore.rules` in this repo) —
   Firestore Database → Rules tab → paste the contents of `firestore.rules` → Publish.
   These are needed for user search and DM permissions to work correctly, even in
   test mode once it expires.

> ⚠️ **Test mode note:** Test mode Firestore rules expire after 30 days and allow anyone
> to read/write. That's fine for an exam demo, but not for a real production app —
> publish `firestore.rules` for correct, scoped access.

### 3. Get dependencies
From the project root:
```bash
flutter pub get
```

### 4. Run the app

This project supports **Android** and **Web (Chrome)**.

**Check available devices:**
```bash
flutter devices
```

**Run on Chrome (fastest for a quick demo, no emulator needed):**
```bash
flutter run -d chrome
```

**Run on Android:**
- Physical phone: enable **Developer Options → USB Debugging**, connect via USB,
  accept the "Allow USB debugging?" prompt on the phone.
- Emulator: open **Android Studio → Device Manager → create/start a virtual device**
  before running `flutter run`.

Then run:
```bash
flutter run -d android
```
or just `flutter run` and pick the Android device from the list if multiple devices show up.

> **"No supported devices connected" / only Windows/Chrome/Edge show up?**
> That means Flutter doesn't see an Android device or emulator yet — it's not
> a code problem. Fix by doing ONE of:
> 1. Plug in a real Android phone with USB debugging enabled, or
> 2. Open Android Studio → Device Manager → start an emulator, then re-run
>    `flutter devices` to confirm it shows up, or
> 3. Just use `flutter run -d chrome` instead — it works immediately with
>    zero extra setup and is enough for an exam demo.

---

## How It Works (for viva/exam explanation)

- **`AuthGate`** in `main.dart` listens to `FirebaseAuth.instance.authStateChanges()`.
  This is a `Stream<User?>` — it automatically emits a new value whenever the user
  logs in or out, so the UI switches between `LoginScreen` and `HomeScreen` without
  any manual navigation code.

- **`AuthService.signUp`** creates the Firebase Auth user, sets their `displayName`,
  and writes a matching profile document to `users/{uid}` in Firestore (with
  lowercase `nameLower`/`emailLower` fields so search works case-insensitively).
  This profile doc is what makes a user *findable* — Firebase Auth itself can't
  be queried by name/email from client code.

- **`ChatService`** is the core of the chat logic:
  - `ensureGlobalChatExists()` — creates the single `chats/global` doc if missing.
  - `searchUsers(query)` — runs two range queries (on `nameLower` and `emailLower`)
    against the `users` collection and merges the results, so a search matches
    either field.
  - `getOrCreateDirectChat(otherUid)` — computes a deterministic chat id by sorting
    both uids together (`"uidA_uidB"`), so two people always land on the *same*
    chat document no matter who starts it first.
  - `streamMyDirectChats()` — real-time list of the current user's DMs, ordered by
    most recent activity.
  - `streamMessages(chatId)` / `sendMessage(chatId, text)` — read/write messages
    under `chats/{chatId}/messages`, used by both the global chat and every DM.

- **`ChatScreen`** is generic — it just takes a `chatId` and a `title`, so the exact
  same screen renders the global chat or any 1-to-1 DM.

- Each message stores: `text`, `senderId`, `senderName`, `senderEmail`, `createdAt`
  (using `FieldValue.serverTimestamp()` for accurate ordering).

---

## Common Setup Issues

| Problem | Fix |
|---|---|
| `Execution failed for task ':app:processDebugGoogleServices'` | Make sure `google-services.json` is inside `android/app/` (already done here) |
| Gradle sync errors | Run `flutter clean` then `flutter pub get` then `flutter run` |
| "Default FirebaseApp is not initialized" | Make sure `await Firebase.initializeApp();` runs before `runApp()` in `main.dart` (already set up) |
| Login/Signup fails with network error | Check your emulator/device has internet access |
| Search returns no results even though the user exists | Make sure `firestore.rules` is published (test mode alone should still work, but published rules are required once it expires); also confirm the target account was created *after* this update, since profile docs are only written on signup |
| "Missing index" error when searching or loading DMs | Firestore may prompt you to create a composite index the first time — click the link in the error message/console log to auto-create it |

---

## Firestore Data Model

**Collection: `users/{uid}`** — public profile for search

| Field | Type | Description |
|---|---|---|
| `name` | string | Display name |
| `email` | string | Email address |
| `nameLower` | string | Lowercased name (for case-insensitive search) |
| `emailLower` | string | Lowercased email (for case-insensitive search) |

**Collection: `chats/{chatId}`** — one global room + one doc per DM

| Field | Type | Description |
|---|---|---|
| `isGlobal` | bool | `true` for the single global chat, `false` for a DM |
| `participants` | array | `[]` for global chat, `[uidA, uidB]` for a DM |
| `lastMessage` | string | Preview text shown on the home screen |
| `lastMessageAt` | timestamp | Used to sort the DM list by recent activity |

**Subcollection: `chats/{chatId}/messages/{messageId}`**

| Field | Type | Description |
|---|---|---|
| `text` | string | Message content |
| `senderId` | string | UID of sender (from Firebase Auth) |
| `senderName` | string | Display name of sender |
| `senderEmail` | string | Email of sender |
| `createdAt` | timestamp | Server-generated timestamp for ordering |

---

Built for a Flutter lab exam — kept intentionally minimal and easy to explain. 🎓
