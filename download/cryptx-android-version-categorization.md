# CryptX MTK Decrypt Inspector - Android Version Categorization

## Overview

This document describes the Android version-based categorization system implemented into CryptX. The system detects the target Android version and applies version-specific validation rules, issue detection, and fix suggestions.

---

## Android Version Mapping

| Version | API Level | Codename | Color | Key Requirements |
|---------|-----------|----------|-------|------------------|
| **Android 11** | 30 | Red Velvet Cake | #4285F4 | TW_INCLUDE_CRYPTO, TW_INCLUDE_CRYPTO_FBE, Keymaster 4.0 |
| **Android 12** | 31 | Snow Cone | #8AB4F8 | Keymaster 4.1, metadata decrypt required |
| **Android 12L** | 32 | Snow Cone v2 | #8AB4F8 | Keymaster 4.1, wrappedkey common |
| **Android 13** | 33 | Tiramisu | #34A853 | fscrypt v2, wrappedkey required, early_hal class |
| **Android 14** | 34 | Upside Down Cake | #EA4335 | Keymaster 4.1, libkeymaster41, fscrypt v2 |
| **Android 15** | 35 | Vanilla Ice Cream | #9C27B0 | Keymint packages, NO metadata decrypt |

---

## Detection Methods (Priority Order)

### 1. PLATFORM_VERSION (Primary)
- Direct parsing of `PLATFORM_VERSION` from BoardConfig.mk
- Handles A15+ workaround pattern `99.x.x`
- Example: `PLATFORM_VERSION := 14.0.0` → Android 14

### 2. PRODUCT_SHIPPING_API_LEVEL (Most Reliable)
- `PRODUCT_SHIPPING_API_LEVEL` from BoardConfig.mk
- `ro.product.first_api_level` from system.prop
- Maps directly to Android API level

### 3. PRODUCT_TARGET_VNDK_VERSION
- `PRODUCT_TARGET_VNDK_VERSION` from BoardConfig.mk
- `ro.vndk.version` from system.prop

### 4. Package Detection (Heuristic)
- Detects `android.hardware.security.keymint` → Android 15
- Detects `android.hardware.keymaster@4.1` → Android 12+

### 5. System Properties
- `ro.build.version.sdk`
- `ro.build.version.preview_sdk`

---

## Version-Specific Requirements

### Android 11 (API 30)
```makefile
# BoardConfig.mk
TW_INCLUDE_CRYPTO := true
TW_INCLUDE_CRYPTO_FBE := true

# Libraries
libkeymaster4, libion

# Service Classes
TEE: class core
Keymaster: class hal
Gatekeeper: class hal
```

### Android 12 (API 31-32)
```makefile
# BoardConfig.mk
TW_INCLUDE_CRYPTO := true
TW_INCLUDE_CRYPTO_FBE := true
TW_INCLUDE_FBE_METADATA_DECRYPT := true
BOARD_USES_METADATA_PARTITION := true

# Libraries
libkeymaster4, libkeymaster41, libion, libxml2

# Service Classes
TEE: class core
Keymaster: class early_hal
Gatekeeper: class hal
```

### Android 13 (API 33)
```makefile
# BoardConfig.mk
TW_INCLUDE_CRYPTO := true
TW_INCLUDE_CRYPTO_FBE := true
TW_INCLUDE_FBE_METADATA_DECRYPT := true

# fscrypt v2+ required in fstab
fileencryption=aes-256-xts:aes-256-cts:v2

# Libraries
libkeymaster4, libkeymaster41, libpuresoftkeymasterdevice, libion, libxml2

# Service Classes
TEE: class core
Keymaster: class early_hal
Gatekeeper: class early_hal
```

### Android 14 (API 34)
```makefile
# BoardConfig.mk
TW_INCLUDE_CRYPTO := true
TW_INCLUDE_CRYPTO_FBE := true
TW_FORCE_KEYMASTER_VER := true  # Recommended

# Libraries (Required)
libkeymaster4, libkeymaster41, libpuresoftkeymasterdevice, libion, libxml2

# Service Classes
TEE: class core
Keymaster: class early_hal
Gatekeeper: class early_hal
```

### Android 15 (API 35)
```makefile
# BoardConfig.mk
TW_INCLUDE_CRYPTO := true
TW_INCLUDE_CRYPTO_FBE := true
TW_FORCE_KEYMASTER_VER := true

# REMOVE these flags (deprecated)
# TW_INCLUDE_FBE_METADATA_DECRYPT := true  # REMOVE
# BOARD_USES_METADATA_PARTITION := true  # REMOVE

# device.mk - Security packages
PRODUCT_PACKAGES += \
    android.hardware.security.keymint \
    android.hardware.security.secureclock \
    android.hardware.security.sharedsecret

# Libraries
libkeymaster4, libkeymaster41, libpuresoftkeymasterdevice, libion, libxml2

# Service Classes
TEE: class core
Keymaster: class early_hal
Gatekeeper: class early_hal
```

---

## New Validation Checks

| Check ID | Category | Description | Weight |
|----------|----------|-------------|--------|
| `android_version` | Version | Android version badge | 3 |
| `ver_flag_*` | Version Flags | Version-specific required flags | 8 |
| `ver_flag_removed_*` | Version Flags | Deprecated flags check | 6 |
| `ver_tee_class` | Version Services | TEE service class | 5 |
| `ver_km_class` | Version Services | Keymaster service class | 7 |
| `ver_gk_class` | Version Services | Gatekeeper service class | 4 |
| `ver_libraries` | Version Libraries | Version-specific libraries | 10 |
| `ver_packages` | Version Packages | Version-specific packages | 8 |
| `ver_km_41` | Version Keymaster | Keymaster 4.1 support (A13+) | 8 |
| `ver_fscrypt_v2` | Version Crypto | fscrypt v2+ policy (A13+) | 5 |
| `ver_wrappedkey` | Version Crypto | wrappedkey + inlinecrypt (A12+) | 7 |

---

## New Issue Types

| Severity | Issue | Probability | Version |
|----------|-------|-------------|---------|
| High | Android X Missing Required Flags | 80% | All |
| Medium | Android X Deprecated Flags Present | 65% | 13+ |
| High | Android X Keymaster Class Incorrect | 75% | All |
| High | Android X Missing Recovery Libraries | 75% | All |
| High | Android X Requires Keymaster 4.1 | 80% | 13+ |
| Medium | Android X Should Use fscrypt v2+ | 60% | 13+ |

---

## UI Enhancements

### Android Version Badge
- Color-coded by Android version
- Shows API level and codename
- Displays detection source

### Verdict Banner
- Added Android version chip alongside TEE chip
- Uses version-specific color

### Overview Tab
- New "Android Version" status card
- Shows name, API level, codename, and detection method

---

## Fix Suggestions by Version

### Android 13+ Fix Template
```rc
service keymaster-4-1 /vendor/bin/hw/android.hardware.keymaster@4.1-service.trustonic
    class early_hal
    interface android.hardware.keymaster@4.0::IKeymasterDevice default
    interface android.hardware.keymaster@4.1::IKeymasterDevice default
    user nobody
    group system drmrpc
    disabled
    seclabel u:r:recovery:s0

on property:hwservicemanager.ready=true
    start keymaster-4-1
```

### Android 15 Security Packages
```makefile
PRODUCT_PACKAGES += \
    android.hardware.security.keymint \
    android.hardware.security.secureclock \
    android.hardware.security.sharedsecret
```

---

## Constants Reference

```javascript
// Android Version to API Level mapping
const ANDROID_VERSIONS = {
  11: { api: 30, name: 'Android 11', codename: 'Red Velvet Cake', color: '#4285F4' },
  12: { api: 31, name: 'Android 12', codename: 'Snow Cone', color: '#8AB4F8' },
  12L: { api: 32, name: 'Android 12L', codename: 'Snow Cone v2', color: '#8AB4F8' },
  13: { api: 33, name: 'Android 13', codename: 'Tiramisu', color: '#34A853' },
  14: { api: 34, name: 'Android 14', codename: 'Upside Down Cake', color: '#EA4335' },
  15: { api: 35, name: 'Android 15', codename: 'Vanilla Ice Cream', color: '#9C27B0' }
};

// Version-specific required flags
const VERSION_FLAGS = {
  11: { required: ['TW_INCLUDE_CRYPTO', 'TW_INCLUDE_CRYPTO_FBE'], optional: ['TW_INCLUDE_FBE_METADATA_DECRYPT'] },
  12: { required: ['TW_INCLUDE_CRYPTO', 'TW_INCLUDE_CRYPTO_FBE', 'TW_INCLUDE_FBE_METADATA_DECRYPT'], optional: ['BOARD_USES_METADATA_PARTITION'] },
  13: { required: ['TW_INCLUDE_CRYPTO', 'TW_INCLUDE_CRYPTO_FBE', 'TW_INCLUDE_FBE_METADATA_DECRYPT'], optional: ['TW_USE_FSCRYPT_POLICY'] },
  14: { required: ['TW_INCLUDE_CRYPTO', 'TW_INCLUDE_CRYPTO_FBE'], optional: ['TW_INCLUDE_FBE_METADATA_DECRYPT', 'TW_USE_FSCRYPT_POLICY', 'TW_FORCE_KEYMASTER_VER'] },
  15: { required: ['TW_INCLUDE_CRYPTO', 'TW_INCLUDE_CRYPTO_FBE'], optional: ['TW_USE_FSCRYPT_POLICY', 'TW_FORCE_KEYMASTER_VER'], removed: ['TW_INCLUDE_FBE_METADATA_DECRYPT', 'BOARD_USES_METADATA_PARTITION'] }
};

// Version-specific library requirements
const VERSION_LIBRARIES = {
  11: ['libkeymaster4', 'libion'],
  12: ['libkeymaster4', 'libkeymaster41', 'libion', 'libxml2'],
  13: ['libkeymaster4', 'libkeymaster41', 'libpuresoftkeymasterdevice', 'libion', 'libxml2'],
  14: ['libkeymaster4', 'libkeymaster41', 'libpuresoftkeymasterdevice', 'libion', 'libxml2'],
  15: ['libkeymaster4', 'libkeymaster41', 'libpuresoftkeymasterdevice', 'libion', 'libxml2']
};

// Version-specific service class requirements
const VERSION_SERVICE_CLASS = {
  11: { tee: 'core', keymaster: 'hal', gatekeeper: 'hal' },
  12: { tee: 'core', keymaster: 'early_hal', gatekeeper: 'hal' },
  13: { tee: 'core', keymaster: 'early_hal', gatekeeper: 'early_hal' },
  14: { tee: 'core', keymaster: 'early_hal', gatekeeper: 'early_hal' },
  15: { tee: 'core', keymaster: 'early_hal', gatekeeper: 'early_hal' }
};
```

---

## Summary

The Android version-based categorization system provides:

1. **Automatic Version Detection** - Multiple detection methods with confidence scoring
2. **Version-Specific Validation** - Different rules per Android version
3. **Version-Specific Issues** - Targeted issue detection based on version requirements
4. **Version-Specific Fixes** - Tailored fix suggestions for the detected version
5. **UI Integration** - Color-coded badges and status cards

This ensures that CryptX provides accurate, relevant analysis based on the specific Android version being targeted by the recovery tree.
