# MelakaTour (WhatNow)

**A Malaysian time-aware lifestyle and travel planning mobile application** focused on exploring Melaka. Built with Flutter for Android (iOS ready) and backed by Google Firebase and Google Maps Platform.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [b. Third-Party Cloud Computing Services](#b-third-party-cloud-computing-services)
- [c. In-House Web Service](#c-in-house-web-service)
- [4. Cloud Services Used (What and Why)](#4-cloud-services-used-what-and-why)
- [5. Design (Architecture, Database, Services, UI)](#5-design-architecture-database-services-ui)
- [Tech Stack](#tech-stack)
- [Setup](#setup)

---

## Overview

**MelakaTour** helps users discover places in Melaka, plan itineraries, check outdoor suitability from live weather, and manage a personalized profile. The app uses **Malaysia time (GMT+8)** for time-based greetings and tips, and integrates **in-house callable web services** (Firebase Cloud Functions) for greetings and weather-based outdoor advice.

---

## Features

- **Explore** — Search and browse places in Melaka (Google Places: Text Search, Nearby). View details, photos, ratings, and open Google Maps for directions.
- **Dynamic Melaka Greeting** — Time-based greeting and travel tips (morning/afternoon/evening/night) from an in-house Cloud Function.
- **Planner** — Create and manage day-by-day itineraries (Firestore). Place autocomplete and details via Google Places; open Google Maps for navigation.
- **Weather / Outdoor Suitability** — GPS-based current weather, 5-day forecast, and an outdoor suitability score (0–100) with recommendations. Uses an in-house Cloud Function that calls Open-Meteo and reverse geocoding; location name reflects the user’s current area.
- **Profile** — Auth-based profile (name, location, activity preferences, profile photo as Base64 in Firestore). Saved places and theme (light/dark) with `SharedPreferences`.
- **Auth** — Email/password sign up, login, forgot password; Firestore-backed user documents.

---

## b. Third-Party Cloud Computing Services

| Provider | Service | What Is Used | For What | Why |
|----------|---------|--------------|----------|-----|
| **Google** | **Firebase Authentication** | Email/password auth, session | Sign up, login, logout, password reset, auth state | Managed auth, no custom server; fits mobile and Firestore security. |
| **Google** | **Cloud Firestore** | NoSQL document DB | `users` (profile, photo as Base64), `saved_places` (per user), itinerary/planner events | Real-time sync, per-user data, optional offline. |
| **Google** | **Firebase Cloud Functions** | Serverless Node.js (callable) | Hosts **in-house** logic: `getMelakaTourTips`, `analyzeMelakaWeather` | Keeps app-specific and weather/greeting logic on our backend; no API keys in the client for Open-Meteo. |
| **Google** | **Firebase (Storage bucket)** | Bucket configured in project | Reserved (e.g. future profile image uploads) | App currently stores profile image as Base64 in Firestore; Storage is available if we move to file uploads. |
| **Google** | **Google Maps Platform** | Maps SDK, Places API, Geocoding, Directions URL | Map (Navigate), place search, nearby, autocomplete, place details, photos, “Open in Google Maps” | Standard for maps, POI search, and navigation in a location-focused app. |
| **Open-Meteo** | **Forecast API** | `api.open-meteo.com/v1/forecast` | Current and 5-day weather (temp, precipitation, WMO codes) | Free, no API key; called **from our Cloud Function** only. |
| **Open-Meteo** | **Geocoding (Reverse)** | `geocoding-api.open-meteo.com/v1/reverse` | Place name from GPS (lat/lon) | Free, no key; used in `analyzeMelakaWeather` to show “Current location” or city/state/country. |

**Note:** Open-Meteo is **not** called directly from the app. It is used only inside the **in-house** Cloud Function `analyzeMelakaWeather`. The app talks only to Firebase (Auth, Firestore, Cloud Functions) and Google Maps/Places.

---

## c. In-House Web Service

The app integrates with **one in-house backend** implemented as **two callable Firebase Cloud Functions** in the project’s Firebase (`whatnow1-6baeb`). Both are HTTP callable (invoked via `FirebaseFunctions.instance.httpsCallable` / `instanceFor(region: 'us-central1')` for `analyzeMelakaWeather`).

### 1. `getMelakaTourTips`

- **Purpose:** Time-based greeting and a single Melaka travel tip.
- **Input:** None (uses server time).
- **Output:** `{ greeting, tip, timestamp }` (e.g. `"Good Morning, Traveler!"`, tip string, ISO timestamp).
- **Logic:** Uses `Asia/Kuala_Lumpur` to get current hour; picks greeting and a random tip from morning/afternoon/evening/night lists.
- **Used by:** `GreetingService` → `HomeVM` → Explore screen (greeting card).
- **Why in-house:** Centralized control over copy and timezone; no third-party dependency for this UX.

### 2. `analyzeMelakaWeather`

- **Purpose:** Outdoor suitability for the user’s **current GPS location** and 5-day forecast.
- **Input:** `{ lat: number, lon: number }` (from the device via `Geolocator`).
- **Output:**  
  `{ suitabilityScore, recommendation, temperature, description, iconCode, locationName, forecast[] }`  
  - `locationName`: from Open-Meteo reverse geocoding (city/state/country) or `"Current location"` if none.
  - `forecast`: 5 entries shaped for `Weather.fromJson` (date, temp, description, icon).
- **Logic:**
  - Calls **Open-Meteo Forecast** and **Open-Meteo Reverse Geocoding** (no API key).
  - Computes a 0–100 suitability score from temperature and precipitation; maps WMO codes to descriptions and OpenWeatherMap-style icon codes for the existing Flutter UI.
- **Used by:** `WeatherService` → `WeatherVM` → Weather screen.
- **Why in-house:** Hides Open-Meteo behind our API, keeps keys and logic on the server, and allows changing the weather provider without app updates.

---

## 4. Cloud Services Used (What and Why)

| Service | What | Why |
|---------|------|-----|
| **Firebase Authentication** | User identity, email/password, session | No custom auth server; integrates with Firestore rules and app flows. |
| **Cloud Firestore** | `users`, `saved_places`, planner/itinerary | Persistent, per-user data; real-time and scalable for a mobile app. |
| **Firebase Cloud Functions** | `getMelakaTourTips`, `analyzeMelakaWeather` | In-house business logic (greetings, weather, suitability) and safe use of Open-Meteo. |
| **Google Maps SDK** | Map in Navigate | Display map and pins for places. |
| **Google Places API** | Text Search, Nearby, Geocoding, Place Details, Autocomplete | Explore search, nearby, planner place search and details. |
| **Google Maps Directions URL** | `https://www.google.com/maps/dir/...` | “Open in Google Maps” from Explore details and Planner. |
| **Open-Meteo (via CF only)** | Forecast + Reverse Geocoding | Free weather and location naming for the Outdoor Suitability feature, without exposing the app to extra endpoints. |

---

## 5. Design (Architecture, Database, Services, UI)

### Architecture: MVVM

- **Models** (`lib/models/`): Data structures and (where applicable) `fromJson` / `toFirestore`:
  - `explore_model`, `greeting_model`, `navigate_model`, `outdoor_suitability_model`, `planner_model`, `trip_model`, `user_model`, `weather_model`
- **Views** (`lib/views/`): UI only; no business logic. They `watch`/`read` ViewModels and call methods (e.g. `fetchWeather`, `fetchPlaces`).
- **ViewModels** (`lib/viewmodels/`): State and logic via `ChangeNotifier`; call Services or Firebase SDK (Auth, Firestore) directly.
- **Services** (`lib/services/`): Encapsulate **callable Cloud Function** calls:
  - `GreetingService` → `getMelakaTourTips`
  - `WeatherService` → `analyzeMelakaWeather` (and GPS via `Geolocator`)

Cloud Functions are **never** called from Views; they go through Service → ViewModel.

### State Management

- **Provider** (`MultiProvider` in `main.dart`): `AuthVM`, `HomeVM`, `ExploreVM`, `NavigateVM`, `WeatherVM`, `PlannerVM`, `SavedPlacesVM`, `ThemeVM`.
- **SharedPreferences**: Theme (light/dark) in `ThemeVM`.

### Database (Firestore)

- **`users/{uid}`**: Profile (name, location, favoriteActivityTypes, photoURL as Base64, etc.).
- **`users/{uid}/saved_places`**: Saved places (e.g. `Place.toFirestore` / `fromFirestore`).
- **Planner / itinerary**: User-scoped collection(s) for itinerary events (date, place, notes, etc.) with `ItineraryEvent.toFirestore` / `fromFirestore`.

Firestore rules and indexes are in `firestore.rules` and `firestore.indexes.json`.

### Services (Backend / External)

- **In-house:** Firebase Cloud Functions `getMelakaTourTips`, `analyzeMelakaWeather` (see [c. In-House Web Service](#c-in-house-web-service)).
- **External (from app):** Firebase Auth, Firestore, Google Maps, Google Places (Text Search, Nearby, Geocoding, Details, Autocomplete), and `url_launcher` for Google Maps directions.

### UI and Navigation

- **Material 3** with light/dark theme; `ThemeVM` and `ThemeData` in `main.dart`.
- **Bottom navigation (MainScreen):** Explore, Planner, Weather, Profile.
- **Screens:** Welcome/Onboarding, Login, SignUp, ForgotPassword, Explore (with greeting card), Explore Details, Planner (add/edit/details), Weather (suitability card + forecast), Profile, Edit Profile, Saved Places, Settings.
- **Libraries:** `intl` (dates), `flutter_native_splash`, `loading_indicator`, `url_launcher` (maps, links).

### Device / Local

- **Geolocator:** GPS for `WeatherService` (outdoor suitability) and location bias in Explore/Navigate.
- **Image picker:** Profile photo (gallery/camera) → Base64 → Firestore (no Firebase Storage in current flow).

---

## Tech Stack

- **Flutter** (Dart 3.x), **Provider**
- **Firebase:** `firebase_core`, `firebase_auth`, `cloud_firestore`, `cloud_functions`
- **Google:** `google_maps_flutter`; Google Places via `http` (REST)
- **Other:** `geolocator`, `flutter_dotenv` (`.env` for `GOOGLE_API`), `shared_preferences`, `image_picker`, `intl`, `url_launcher`

---
(IGNORE THIS)
## Setup

1. **Clone and install**
   ```bash
   flutter pub get
   ```

2. **Environment**
   - Add a `.env` with `GOOGLE_API=<your-Google-Maps-API-key>` (Maps + Places enabled).
   - Ensure `lib/firebase_options.dart` and `android/app/google-services.json` match your Firebase project (e.g. `whatnow1-6baeb`).

3. **Firebase**
   - Create/use a project in [Firebase Console](https://console.firebase.google.com).
   - Enable Auth (Email/Password), Firestore, and Functions; deploy:
     ```bash
     firebase deploy --only functions
     ```

4. **Run**
   ```bash
   flutter run
   ```
   (Android is the primary target; iOS would need `firebase_options` and `GoogleService-Info.plist` configured.)

---

## License

Proprietary / project use.
