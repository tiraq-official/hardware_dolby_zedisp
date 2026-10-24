# daxService bridge — make Dolby profiles actually change the sound

## Why this is needed

`LunarisDolby` sets the profile correctly — the probe proved `setIntParam(0xA000000, n)`
works and reads back right. But in **Motorola's** stack that only records *which*
profile is selected. The per-profile **tuning** (what makes Dynamic, Movie, Music and
Spatial Audio sound different) is loaded by the **DMS**, and the thing that tells the
DMS to load it is `daxService`.

Decompiling `daxService.apk` showed the exact sequence in
`DaxService.onSetDolbyProfile()`:

```java
DolbyAudioEffect.setProfile(n)                          // ← Lunaris already does this
DolbyAudioEffectHelper.updateTuningDeviceIfNeeded(...)  // ← Lunaris never did
```

`updateTuningDeviceIfNeeded` drives the DMS's `setSelectedTuningDevice()`. Without it
the effect's parameter cache commits 0 changes — the `Committed 0 parameters out of 66`
you saw — so every profile sounds identical.

`daxService`'s `ContentProvider` (`com.dolby.daxservice.dolbyprovider`) takes the profile
as the **last path segment** of `…/3/profile/<n>`, and its `update()` runs the whole
sequence, tuning device included.

## What changed

| file | change |
|---|---|
| `proprietary/system_ext/priv-app/daxService/daxService.apk` | new — from the fogos stock dump |
| `Android.bp` | new `android_app_import` module `daxService` (privileged, platform-signed, system_ext) |
| `dolby.mk` | `daxService` in `PRODUCT_PACKAGES` + the permission XML copy |
| `configs/permissions/com.dolby.daxservice.xml` | new — daxService's privileged permissions |
| `configs/permissions/privapp-permissions-dolby.xml` | `DOLBY_UPDATE_BROADCAST` for `org.lunaris.dolby` |
| `LunarisDolby/AndroidManifest.xml` | `<uses-permission …DOLBY_UPDATE_BROADCAST>` |
| `LunarisDolby/…/data/DolbyRepository.kt` | new `mirrorProfileToDaxService()`, called on profile switch and on restore |

The mirror call is one line of real work:

```kotlin
context.contentResolver.update(
    Uri.parse("content://com.dolby.daxservice.dolbyprovider/3/profile/$profile"),
    ContentValues(), null, null)
```

It's wrapped in try/catch, so if the permission is refused it logs and moves on — the
app cannot crash from this.

## Apply

```bash
cd /path/to/hardware_dolby_zedisp
unzip -o ~/Downloads/dolby-daxservice-bridge.zip

git status --short
git add -A
git commit -m "dolby: drive profiles through daxService so the DMS loads the tuning"
git push
```

You should see exactly 7 entries (6 modified + the new APK and XMLs).

## Verify after flashing

Switch profiles in the app and watch:

```bash
adb logcat -s DolbyProvider DolbyAudioEffectHelper DlbDapEndpointParamCache
```

Success looks like:

```
DolbyProvider: update() set dolby profile to 4
DolbyAudioEffectHelper: setDolbyProfile() ...
DlbDapEndpointParamCache: doCommit() Committed 40 parameters out of 66      ← NOT 0
```

A non-zero commit count means the DMS loaded the profile's tuning — and that's the
audible change. Then A/B **Spatial Audio** against **Dynamic** on headphones.

## If the permission is refused

`com.dolby.permission.DOLBY_UPDATE_BROADCAST` is declared at `signature` level by
`daxService`, and the stock APK carries a `knownCerts` restriction pointing at
Motorola's platform certificate. We re-sign `daxService` with the platform key
(`certificate: "platform"`), which normally makes it match — but if the mirror call
logs a `SecurityException`, the fix is to declare the permission ourselves so the
platform key owns it. Tell me and I'll add that; the log line will say
`daxService mirror failed for profile N: ...`.

## Note

`DAXBRIDGE-README.md` is documentation for you, not a repo file — delete it before
committing if you'd rather keep the tree clean.
