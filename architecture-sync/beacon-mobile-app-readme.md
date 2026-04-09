# beacon-mobile-app

Flutter-based mobile application for the Beacon emergency-response platform. Part of the [Beacon](https://beacon-tcd.tech) platform.

Available for Android and iOS. Serves two distinct user groups on the same codebase: **citizens** who report and track emergencies, and **ground staff** who respond to dispatches in the field. The app's available features and navigation structure adapt based on the authenticated user's role.

---

## Overview

The mobile app is the primary interface for citizens affected by city-scale emergencies. A citizen can report an incident from their current location with a few taps, view a live map of ongoing incidents with severity-coded markers, receive push notifications when new incidents are approved near them, and get real-time evacuation routing that directs them away from the emergency. They can also access live safety information — hospital availability, weather warnings, flood levels — and connect with an AI-powered support bot if they are distressed.

Ground staff use the same app with additional operational features: receiving and acknowledging dispatch assignments, navigating TO an incident site, communicating in team channels, toggling their on-duty status, and viewing their dispatch history.

---

## Tech Stack

| Layer | Package / Tool | Purpose |
|---|---|---|
| Framework | Flutter 3.9+ / Dart 3.x | Cross-platform UI |
| State management | Riverpod (`flutter_riverpod`, `riverpod_annotation`) | Async providers, `AsyncNotifier`, `family` providers |
| Routing | `go_router` | Declarative routing, deep links, auth + role guards |
| HTTP | `dio` | Interceptor-based HTTP with auth token injection and retry |
| Models | `freezed` + `json_serializable` | Immutable, auto-generated Dart models |
| Maps | `flutter_map` + TomTom tiles | Map rendering; TomTom Routing API for navigation |
| Push notifications | `firebase_messaging` + `flutter_local_notifications` | FCM push + local notification scheduling |
| OAuth | `google_sign_in` + `flutter_web_auth_2` | Citizen Google sign-in |
| Secure storage | `flutter_secure_storage` | JWT access + refresh tokens |
| Preferences | `shared_preferences` | Non-sensitive user preferences |
| Connectivity | `connectivity_plus` | Offline detection and retry |
| Logging | `logger` | Debug-only structured logging (never `print()`) |

---

## Architecture

The app follows Clean Architecture on a per-feature basis. Each feature module is self-contained with its own data, domain, and presentation layers:

```
features/<feature>/
  data/             # Repository implementations, DTOs, data sources
  domain/           # Abstract repository interfaces, Freezed entity models
  presentation/
    providers/      # Riverpod providers (AsyncNotifier, family, StreamProvider)
    pages/          # Full screen widgets — own data fetching, compose components
    components/     # Feature-specific UI (cards, lists, detail panels)
    atoms/          # Smallest reusable units (badges, status chips)
```

**Widget hierarchy: Page → Component → Atom**

Pages own data fetching through Riverpod providers. They compose components that receive data as constructor parameters. Atoms are pure display widgets with no business logic. This structure means any screen can be navigated to directly by URL without depending on in-memory state from a parent widget.

### Shared Core

Cross-feature infrastructure lives in `lib/core/`:

```
core/
  config/       # EnvConfig (reads --dart-define at compile time)
  data/api/     # API service classes (iam_api, core_api, observability_api, tomtom_api, etc.)
  data/dto/     # Shared Freezed DTOs with @JsonSerializable
  error/        # Failure sealed class (ServerFailure, NetworkFailure, AuthFailure, CacheFailure)
  network/      # api_client.dart (Dio instance + interceptors), auth_interceptor.dart
  router/       # app_router.dart, route_guards.dart, route_names.dart
  theme/        # BeaconColors, BeaconTypography, BeaconSpacing, BeaconRadius, app_theme.dart
  utils/        # severity_utils, incident_type_utils, location_utils, date_utils
  widgets/      # beacon_button, error_view, empty_state, skeleton_loader
```

---

## Features

### Authentication (`features/auth`)

**Citizens** sign in via Google OAuth. The app launches a browser-based OAuth consent flow using `flutter_web_auth_2` with the `beaconflutterapp` custom URL scheme as the callback. After successful authorization, the app exchanges the auth code with beacon-iam and stores the JWT token pair in `FlutterSecureStorage`.

**Ground staff** sign in with username and password through beacon-iam's credential endpoint. Registration is also available directly from the login screen.

Both flows are handled by `AuthNotifier`: `loginWithGoogle()`, `loginWithCredentials()`, and `registerStaff()`. The stored tokens are injected on every subsequent request by `AuthInterceptor`, which also handles silent token refresh when the access token is within 60 seconds of expiry.

### Incidents (`features/incidents`)

The citizen home screen shows a paginated list of recent approved incidents near the user's location, with severity-coloured markers, incident type icons, and timestamps. Tapping an incident opens a detail screen with the full description, current status, active dispatches summary, and any citizen-visible updates posted by the incident commander.

Citizens can report new incidents from the "+" button. The report form captures incident type (fire, flood, structural damage, medical, SOS, other), a description, and automatically includes the device's current location. Photo attachments can be uploaded to the incident vault.

SOS incidents are a special case: the SOS button on the home screen creates an incident immediately from the user's current location with the highest priority, without requiring the user to fill in a form.

### Incident Map (`features/map`)

An interactive map powered by `flutter_map` with TomTom raster tiles. Incident markers are colour-coded by severity (P0 critical = red, P1 high = orange, P2 medium = yellow, P3 low = blue) and clustered at lower zoom levels. Tapping a marker opens the incident detail panel.

The map supports two navigation modes:
- **Ground staff navigation**: Routes TO the incident site using the TomTom Routing API (`calculateRoute`). Shows turn-by-turn waypoints and estimated arrival time.
- **Citizen evacuation routing**: Routes AWAY from the incident to the nearest designated assembly point using `calculateEvacuationRoute`. This is automatically suggested when a citizen is within an active evacuation zone.

Both routing modes display the route polyline on the map. An "Open in Maps" button hands off to the device's native navigation app (Google Maps or Apple Maps) for hands-free turn-by-turn navigation during active response.

If `TOMTOM_API_KEY` is not configured (local debug builds), the map falls back to Carto CDN tiles and routing is disabled.

### Safety Dashboard (`features/safety`)

Real-time safety information sourced from beacon-observability:
- **Weather**: Current conditions and warnings for the user's location
- **Hospitals**: Nearest open hospitals with ED wait times and diversion status
- **Shelters**: Active emergency shelters with capacity and address
- **Evacuations**: Active evacuation zones with boundaries and instructions
- **Traffic**: Major road closures and incidents affecting movement
- **Floods**: Water level sensors with flood risk classification
- **Power**: Active power outages in the area

Each category is a card on the safety dashboard screen. Tapping a card opens a detail screen with the full list. High-severity items (active evacuations, flood warnings, severe weather) are surfaced at the top of the screen regardless of category.

### Push Notifications (`features/notifications`)

The app registers the device's FCM token with beacon-iam on startup and whenever Firebase rotates the token. Incoming push notifications are displayed as system notifications when the app is in the background and as in-app banners when it is in the foreground.

The notification list screen shows all past notifications with read/unread status. Tapping a notification navigates to the relevant incident or evacuation detail screen. Notifications can be dismissed individually or cleared all at once.

### AI Support Bot Chat (`features/chat`)

A conversational interface to beacon-support-bot. Citizens can open a chat session when they are distressed or need guidance during an emergency. The bot provides calm, contextually-aware responses generated by Vertex AI's Gemini model, grounded with information about the specific incident the user is affected by.

The chat screen also provides a direct SOS button that creates an emergency incident immediately and simultaneously escalates the conversation to a human support agent.

### Ground Staff Features (`features/ground_staff`)

Ground staff users see an additional tab in the bottom navigation bar with their operational tools:

**Dashboard**: A summary of the staff member's current duty status, any pending dispatch assignments, and their recent dispatch history.

**Dispatches**: The list of active and pending dispatch assignments. Each dispatch shows the incident details, estimated distance, priority, and the accept/reject action buttons. Accepting a dispatch starts a timer and marks the staff member as responding. Rejecting requires a brief reason.

**Team Chat**: Access to the team communication channels from beacon-communication-platform. Ground staff use this to coordinate with other responders and the dispatch operator.

**Navigation**: When a dispatch is accepted, the navigation screen launches automatically with a TomTom-routed path to the incident location.

**Duty Toggle**: A prominent on-duty / off-duty toggle on the dashboard. When off-duty, the staff member is excluded from the dispatch availability pool and will not receive new dispatch assignments.

### User Profile (`features/profile`)

The profile screen shows the user's name, email, role, active sessions (with the ability to sign out other devices), registered devices and notification preferences, and a log of recent activity.

Ground staff can view their dispatch history and performance statistics. Citizens can view their incident report history and update their emergency contact information.

---

## Building and Running

### Prerequisites

| Tool | Version |
|---|---|
| Flutter | 3.9+ |
| Dart | 3.x (SDK `^3.9.2`) |
| Android Studio / Xcode | Latest stable |
| `flutter_pub` | Included with Flutter |

### Setup

```bash
# Get dependencies
flutter pub get

# Generate Freezed models and Riverpod annotations
dart run build_runner build --delete-conflicting-outputs

# Verify no errors
flutter analyze        # must pass with zero errors
flutter test           # all tests must pass
```

### Running in Debug Mode

```bash
# Android
flutter run --dart-define=BASE_URL=https://beacon-tcd.tech \
            --dart-define=TOMTOM_API_KEY=your-key

# iOS (requires macOS + Xcode)
flutter run --dart-define=BASE_URL=https://beacon-tcd.tech \
            --dart-define=TOMTOM_API_KEY=your-key
```

In debug mode, `BASE_URL` falls back to `https://beacon-tcd.tech` if not provided. `TOMTOM_API_KEY` falls back to empty, which switches map tiles to Carto CDN and disables routing.

### Building for Production

```bash
# Android APK
flutter build apk \
  --dart-define=BASE_URL=https://beacon-tcd.tech \
  --dart-define=TOMTOM_API_KEY=your-tomtom-key

# Android App Bundle (for Play Store)
flutter build appbundle \
  --dart-define=BASE_URL=https://beacon-tcd.tech \
  --dart-define=TOMTOM_API_KEY=your-tomtom-key

# iOS
flutter build ios \
  --dart-define=BASE_URL=https://beacon-tcd.tech \
  --dart-define=TOMTOM_API_KEY=your-tomtom-key
```

**Never hardcode `BASE_URL` or `TOMTOM_API_KEY` in Dart source files.** Use `EnvConfig.baseUrl` and `EnvConfig.tomTomApiKey` wherever these values are needed.

### Environment Variables

| `--dart-define` | Required | Description |
|---|---|---|
| `BASE_URL` | Yes (release) | Beacon backend API base URL |
| `TOMTOM_API_KEY` | Yes (release) | TomTom Maps API key for tile rendering and routing |

---

## Firebase Setup

The app uses Firebase Cloud Messaging for push notifications. Before building:

1. Create a Firebase project and add Android (`com.beacon.app`) and iOS (`com.beacon.app`) apps
2. Download `google-services.json` (Android) to `android/app/`
3. Download `GoogleService-Info.plist` (iOS) to `ios/Runner/`
4. The `firebase_options.dart` file in `lib/` is auto-generated by `flutterfire configure`

The Firebase project must have the Cloud Messaging API enabled. The server key from the Firebase project is registered with beacon-iam as the FCM sender credential.

---

## API Domains

All API calls go through the Dio client configured in `core/network/api_client.dart`. The base URL is set from `EnvConfig.baseUrl`. The `AuthInterceptor` injects the JWT token and handles silent refresh.

| Domain | Base Path | Service |
|---|---|---|
| Authentication | `/api/iam/v1/auth` | beacon-iam |
| User / Profile | `/api/iam/v1` | beacon-iam |
| Incidents | `/api/core/v1` | beacon-core |
| Notifications | `/api/notification/v1` | beacon-notification |
| Communication | `/api/communication/v1` | beacon-communication-platform |
| Support Bot | `/api/support-bot/v1` | beacon-support-bot |
| Observability | `/api/observability/v1` | beacon-observability |
| TomTom (tiles) | `https://api.tomtom.com/map/1/tile/` | TomTom (external) |
| TomTom (routing) | `https://api.tomtom.com/routing/1/` | TomTom (external) |

TomTom API calls use their own dedicated Dio instance (`tomTomApiProvider`) without the auth interceptor — TomTom uses its own API key authentication passed as a query parameter.

---

## Project Structure

```
lib/
├── core/
│   ├── config/             # EnvConfig — reads --dart-define values
│   ├── data/api/           # iam_api, core_api, observability_api, tomtom_api, etc.
│   ├── data/dto/           # Shared Freezed DTOs
│   ├── error/              # Failure sealed class
│   ├── network/            # api_client.dart, auth_interceptor.dart
│   ├── router/             # app_router.dart, route_guards.dart, route_names.dart
│   ├── theme/              # BeaconColors, typography, spacing, radius, app_theme.dart
│   ├── utils/              # severity_utils, date_utils, location_utils
│   └── widgets/            # beacon_button, error_view, empty_state, skeleton_loader
├── features/
│   ├── auth/               # Login (Google + credentials), registration
│   ├── incidents/          # Feed, detail, reporting, SOS
│   ├── map/                # Interactive map, markers, navigation
│   ├── safety/             # Weather, hospitals, shelters, evacuations, traffic, floods, power
│   ├── notifications/      # Push notification list, mark read
│   ├── chat/               # AI support bot conversation, SOS button
│   ├── profile/            # User profile, sessions, devices, activity
│   └── ground_staff/       # Dashboard, dispatches, team chat, duty toggle
├── shared/
│   ├── atoms/              # beacon_chip, info_row, status_dot, count_badge, tag_chip
│   ├── components/         # section_header, paginated_list, pull_to_refresh_list
│   └── providers/          # location_provider, connectivity_provider, theme_provider
├── services/
│   ├── notification_service.dart   # FCM token registration, local notification handling
│   └── device_service.dart         # Device info for beacon-iam registration
├── firebase_options.dart   # Auto-generated Firebase config
└── main.dart               # App entry point, ProviderScope, app_theme setup
```
