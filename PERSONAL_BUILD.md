# Immich Homelab Android build

Personal, AI-assisted fork based on official Immich v3.2.4 (`db355f79d910bbfc6378117ed10868493c97b922`). This is not an official Immich release or an upstream pull request.

## Fix

Manually selected uploads calculate a missing local SHA-1/Base64 checksum using a file stream. The checksum is saved only after a successful upload, provided file metadata and database edit markers are unchanged. Normal server sync then associates the local file with the current account's server asset and enables the existing cloud indicator. Failed/cancelled uploads do not receive the association. Existing checksums are preserved. No album backup selection is changed and no database schema migration is introduced.

The extra read consumes I/O and CPU, especially for videos. Memory use is bounded by streaming and the existing worker pool limits concurrency. Cancellation is checked before and after hashing. File metadata checks reduce edit races but are not an atomic snapshot of the uploaded bytes. Device performance and the actual icon refresh must still be checked on a phone.

## Test evidence

78 tests passed across foreground upload, local asset repository, upload actions, background hash service, and timeline repository suites. Analysis of changed Dart files passed. The new manual-upload persistence regression fails when the manual checksum path is disabled and passes with the fix enabled. This verifies code behavior; it does not replace device testing.

## Separate app

With `IMMICH_PERSONAL_BUILD=1`, the release app uses package `app.alextran.immich.homelab` and label **Immich Homelab**. It requires an explicit personal signing key rather than falling back to debug signing. Widget launch intents are scoped to this app. Its OAuth callback scheme is `app.immich.homelab` and must be passed to Dart as well; OAuth providers would need the corresponding redirect URI configured. Password login needs no OAuth change.

The official Play Store app stays installed. This app has its own login and local database. Do not enable two clients to perform automatic backup until you have tested the personal build. Generic `immich://` links can still offer both apps; official verified HTTPS links remain handled by the official app, and this build does not claim automatic link verification.

## Build

Use the release's pinned tools: Flutter 3.47.1, Java 21, OpenAPI Generator 7.25.0, and the Android SDK/NDK requested by Flutter. Install and generate according to `mobile/mise.toml` and the official developer documentation. On Windows, use Git with `core.longpaths=true` and check that Git dependencies really checked out their pinned commit after any failed download. Keep Java's Unix-domain socket temporary directory short to avoid Windows loopback errors.

This patch does not change schema. Generate migration steps from the recorded schema with:

```text
dart run drift_dev schema steps drift_schemas/main lib/data/db/main/database.steps.dart
```

The normal OpenAPI, translations, Pigeon and build_runner outputs are still required. Do not regenerate/commit a different database schema simply to bypass a host formatting mismatch.

Run from `mobile/`:

```text
flutter test test/services/foreground_upload.service_test.dart test/medium/repositories/local_asset_repository_test.dart
flutter test test/unit/presentation/actions/upload_action_test.dart test/unit/services/hash_service_test.dart test/medium/repositories/timeline_repository_test.dart
```

For a personal ARM64 release, set these environment variables in the build process:

```text
IMMICH_PERSONAL_BUILD=1
IMMICH_PERSONAL_KEYSTORE=<absolute-path-to-private-keystore>
IMMICH_SIGNING_PASSWORD=<private-password>
```

The signing alias is `immich-personal`. The Gradle configuration currently uses the same password for store and key. Never commit the keystore, passwords, or an environment file containing them. Build:

```text
flutter build apk --release --target-platform android-arm64 --dart-define=IMMICH_OAUTH_SCHEME=app.immich.homelab --build-name=3.2.4 --build-number=3020401
```

Retain the same signing key and application ID, and increase the build number for each later personal APK. Keep a secure recovery copy of the signing key and password; losing them prevents updates to the installed personal app. Android downgrades and local database migrations are not guaranteed reversible.

## Phone verification

1. Install **Immich Homelab** alongside Immich and log into your server. Leave automatic backup disabled initially.
2. Use an album not selected for backup. Select two or three sample photos and tap Upload.
3. Let normal server sync finish. If needed, leave/reopen the album. Verify only the selected photos have a cloud checkmark, and confirm the server has those photos.
4. Close/reopen the app and repeat the check. Upload an already-existing sample to check duplicate handling.
5. Test a cancelled upload, a network failure, an edited sample photo and a video. Verify unsuccessful uploads are not falsely marked as backed up.
6. Check the original whole-album backup workflow and monitor responsiveness and battery usage before adopting it for your full library.

Previously uploaded photos with missing local hashes are not automatically repaired. Re-selecting a sample for Upload should allow server deduplication while saving the local hash; verify this on your phone before doing it in bulk.

## Updating from upstream

Keep `main` as the upstream tracking branch and personal releases on separate branches. Fetch stable release tags from `https://github.com/immich-app/immich.git`, create a new branch at a chosen server-compatible tag, and apply the small fix and packaging commits separately. Review conflicts, regenerate using that release's tools, run the suites above and phone checks, and build/sign a new APK. Do not automatically install merged development snapshots. Remove the fix when upstream offers equivalent behavior.

The functional change consists of the foreground upload service, local asset repository, and their two test files. Personal Android packaging consists of Gradle, the manifest, OAuth callback configuration, and widget intent scoping. Keep these concerns separate when updating.

Upstream currently asks contributors not to open LLM-generated PRs. This AI-assisted change remains in this personal fork. Disclose its origin and ask maintainers for guidance before submitting upstream.
