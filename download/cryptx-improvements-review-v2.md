# CryptX MTK Decrypt Inspector - Complete Improvement Review
## Based on Commit 97f07ab (Tecno LH8n TWRP - Fix Decryption on A15+)

---

## 📊 Executive Summary

**Commit:** `97f07abfdaf57e0b5ed7395eee74a0007eb02e0b`  
**Message:** "LH8n : fix decryption on A15+"  
**Author:** naden01 (nazephyrus)  
**Date:** 2026-02-19  
**Stats:** +542 lines, -1711 lines | 11 files changed  

**Device:** Tecno LH8n (mt6833 platform, Trustonic TEE)  
**Result:** Successfully boots and decrypts on Android 15+

This commit provides **critical patterns** for A15+ decryption that CryptX should detect and validate.

---

## 🔍 Complete File-by-File Analysis

### 1. BoardConfig.mk Changes

#### 1.1 Crypto Configuration Changes

```diff
- TW_INCLUDE_CRYPTO               := true
- TW_INCLUDE_CRYPTO_FBE           := true
- TW_INCLUDE_FBE_METADATA_DECRYPT := true    # REMOVED
- BOARD_USES_METADATA_PARTITION   := true    # REMOVED
- TW_USE_FSCRYPT_POLICY           := 2
- TW_FORCE_KEYMASTER_VER          := true
+ TW_INCLUDE_CRYPTO          = true          # Changed from := to =
+ TW_INCLUDE_CRYPTO_FBE  := true
+ TW_USE_FSCRYPT_POLICY  := 2
+ TW_FORCE_KEYMASTER_VER := true
```

**Critical Insight:** 
- `TW_INCLUDE_FBE_METADATA_DECRYPT` and `BOARD_USES_METADATA_PARTITION` were **REMOVED**
- This is **NOT an error** - it's required for A15+ devices
- Metadata partition handling changed in Android 15

#### 1.2 Platform Version Workaround

```diff
- PLATFORM_VERSION                := 14
+ PLATFORM_VERSION             := 99.87.36
```

**Critical Insight:**
- Platform version `99.x.x` is an intentional A15+ compatibility workaround
- TWRP's decryption logic uses this to determine API behavior
- Should **NOT** be flagged as unusual

#### 1.3 New Flag Added

```diff
+ BOARD_SUPPRESS_SECURE_ERASE := true
```

**Purpose:** Prevents secure erase operations that may fail in recovery on some eMMC/UFS storage

---

### 2. device.mk Changes

#### 2.1 A/B OTA Partition Updates

```diff
  AB_OTA_PARTITIONS += \
      boot \
      dtbo \
      lk \
      product \
      system \
      system_ext \
+     vbmeta \                    # ADDED
      vbmeta_system \
      vbmeta_vendor \
      vendor \
      vendor_boot
```

#### 2.2 Filesystem Type Change

```diff
  AB_OTA_POSTINSTALL_CONFIG += \
      RUN_POSTINSTALL_system=true \
      POSTINSTALL_PATH_system=system/bin/mtk_plpath_utils \
-     FILESYSTEM_TYPE_system=erofs \
+     FILESYSTEM_TYPE_system=ext4 \
      POSTINSTALL_OPTIONAL_system=true

  AB_OTA_POSTINSTALL_CONFIG += \
      RUN_POSTINSTALL_vendor=true \
      POSTINSTALL_PATH_vendor=bin/checkpoint_gc \
-     FILESYSTEM_TYPE_vendor=erofs \
+     FILESYSTEM_TYPE_vendor=ext4 \
      POSTINSTALL_OPTIONAL_vendor=true
```

**Critical Insight:** Changed from `erofs` to `ext4` for postinstall - EROFS may not be readable in recovery without proper kernel support

#### 2.3 Boot Control Changes

```diff
  PRODUCT_PACKAGES += \
-     android.hardware.boot@1.2-service \      # REMOVED
      android.hardware.boot@1.2-mtkimpl \
      android.hardware.boot@1.2-mtkimpl.recovery

  PRODUCT_PACKAGES_DEBUG += \
-     bootctrl
+     bootctl                                    # Changed name

- PRODUCT_PACKAGES += \
-     bootctrl.mt6833 \                         # REMOVED
-     bootctrl.mt6833.recovery                  # REMOVED
```

#### 2.4 Removed DRM, Added Security HALs

```diff
- # Drm
- PRODUCT_PACKAGES += \
-     android.hardware.drm@1.4                  # REMOVED

+ # Security
+ PRODUCT_PACKAGES += \
+     android.hardware.security.keymint \       # ADDED
+     android.hardware.security.secureclock \   # ADDED
+     android.hardware.security.sharedsecret    # ADDED
```

**Critical Insight:** Keymint packages are **required** for A15+ devices

#### 2.5 Library Configuration Change

```diff
- # Additional Libraries
- TARGET_RECOVERY_DEVICE_MODULES += \
-     android.hardware.keymaster@4.1 \
-
- RECOVERY_LIBRARY_SOURCE_FILES += \
-     $(TARGET_OUT_SHARED_LIBRARIES)/android.hardware.keymaster@4.1

+ # Additional configs
+ TW_RECOVERY_ADDITIONAL_RELINK_LIBRARY_FILES += \
+     $(TARGET_OUT_SHARED_LIBRARIES)/android.hardware.keymaster@4.1
+
+ TARGET_RECOVERY_DEVICE_MODULES += \
+     android.hardware.keymaster@4.1
```

#### 2.6 Added OTA Support Packages

```diff
+ PRODUCT_PACKAGES += \
+     otapreopt_script \
+     cppreopts.sh
```

---

### 3. init.recovery.mt6833.rc Changes

#### 3.1 Import Changes

```diff
- import /tee.rc
- import /trustonic.rc
+ import /init.tee.rc
  import /init.custom.rc
```

**Critical Insight:** Merged `tee.rc` + `trustonic.rc` into single `init.tee.rc` (follows Android naming convention)

#### 3.2 USB Configuration

```diff
  on init
      setprop sys.usb.configfs 1
      setprop sys.usb.controller "musb-hdrc"
-     setprop sys.usb.ffs.aio_compat 1
+     setprop sys.usb.ffs.aio_compat 0
-     export LD_LIBRARY_PATH /system/lib64:/vendor/lib64:/vendor/lib64/hw:/system/lib64/hw
+     export LD_LIBRARY_PATH /system/lib64:/vendor/lib64:/vendor/lib64/hw
      setprop crypto.ready 1

+ on fs && property:ro.debuggable=0
+     write /sys/class/udc/musb-hdrc/device/cmode 2
+     start adbd
```

#### 3.3 Boot Device Detection (NEW)

```diff
  on fs
      install_keyring
+     wait /dev/block/platform/soc/11270000.ufshci
+     symlink /dev/block/platform/soc/11270000.ufshci /dev/block/bootdevice
```

**Critical Insight:** Added UFS device detection and symlink - essential for proper block device access

#### 3.4 Preloader Symlink Timing

```diff
+ on post-fs
+     start boot-hal-1-2
+
+     symlink /dev/block/mapper/pl_a /dev/block/by-name/preloader_raw_a
+     symlink /dev/block/mapper/pl_b /dev/block/by-name/preloader_raw_b

-     on property:persist.vendor.mtk.pl_lnk=1
-     symlink /dev/block/mapper/pl_a /dev/block/by-name/preloader_raw_a
-     symlink /dev/block/mapper/pl_b /dev/block/by-name/preloader_raw_b
```

**Critical Insight:** Moved preloader symlinks from property trigger to `post-fs` - earlier execution

#### 3.5 mtk_plpath_utils Path Change

```diff
- service mtk.plpath.utils.link /vendor/bin/mtk_plpath_utils
+ service mtk.plpath.utils.link /system/bin/mtk_plpath_utils
      class main
      user root
      group root system
-     disabled
      oneshot
+     disabled
      seclabel u:r:recovery:s0
```

**Critical Insight:** Changed from `/vendor/bin/` to `/system/bin/` - system partition is more reliable in recovery

#### 3.6 Gatekeeper Service Changes

```diff
  service vendor.gatekeeper-1-0 /vendor/bin/hw/android.hardware.gatekeeper@1.0-service
      interface android.hardware.gatekeeper@1.0::IGatekeeper default
-     class hal
      user root
      group root
      disabled
      seclabel u:r:recovery:s0
```

#### 3.7 Keymaster Service Changes (CRITICAL)

```diff
  service vendor.keymaster-4-1-trustonic /vendor/bin/hw/android.hardware.keymaster@4.1-service.trustonic
-     user root
-     group root drmrpc
-     disabled
-     oneshot
+     class early_hal
+     interface android.hardware.keymaster@4.0::IKeymasterDevice default
+     interface android.hardware.keymaster@4.1::IKeymasterDevice default
+     user nobody
      seclabel u:r:recovery:s0
```

**Critical Changes:**
1. **Added `class early_hal`** - Key HALs should use this class for proper lifecycle
2. **Added explicit interface declarations** - Required for HAL registration
3. **Changed `user root` to `user nobody`** - More secure, less privilege
4. **Removed `oneshot`** - Service managed by HAL class
5. **Removed `group root drmrpc`** - Not needed with proper SELinux context

#### 3.8 Service Start Order Changes

```diff
  on property:crypto.ready=1
-     start mobicore
-     start keymaster_attestation-1-1
-     start vendor.gatekeeper-1-0
      start vendor.keymaster-4-1-trustonic

  on property:hwservicemanager.ready=true
      start mobicore
-     start keymaster_attestation-1-1
      start vendor.gatekeeper-1-0
      start vendor.keymaster-4-1-trustonic
```

**Critical Insight:** 
- Removed `keymaster_attestation-1-1` service entirely
- `crypto.ready=1` now only starts keymaster (mobicore starts after hwservicemanager)

#### 3.9 Added Vendor Data Directories

```diff
+ on post-fs-data
+     mkdir /data/vendor_de 0770 system system
+     mkdir /data/vendor_de/0 0770 system system
+     mkdir /data/vendor_de/0/cryptoeng 0770 system system
+
+     start health-hal-2-1
+     setprop sys.usb.config adb
```

#### 3.10 USB Mode Change

```diff
- on boot
-     start health-hal-2-1
-     setprop sys.usb.config mtp,adb
-     setprop sys.usb.state mtp,adb
-     setprop persist.sys.usb.config mtp,adb
```

**Changed:** MTP+ADB → ADB only (simpler, more reliable)

---

### 4. init.tee.rc (NEW FILE - Merged Configuration)

**Combined from:** `tee.rc` + `trustonic.rc`

#### 4.1 mcDriverDaemon Service Configuration

```
service mobicore /vendor/bin/mcDriverDaemon --sfs-reformat --P1 /mnt/vendor/persist/mcRegistry \
    -r /vendor/app/mcRegistry/06090000000000000000000000000000.drbin \
    -r /vendor/app/mcRegistry/020f0000000000000000000000000000.drbin \
    ... (26 total drbin files)
    -r /vendor/app/mcRegistry/5020170115e016302017012521300000.drbin
    user root
    group root
    disabled
    seclabel u:r:recovery:s0
```

#### 4.2 Added drbin (NEW in this commit)

```diff
+ -r /vendor/app/mcRegistry/070c0000000000000000000000000000.drbin \
```

**This drbin was missing in the old trustonic.rc!** - It handles IRIS_GPIO / DrTui

#### 4.3 drbin Function Mapping

| drbin File | Function |
|------------|----------|
| `06090000000000000000000000000000.drbin` | DRM keyinstall (CRITICAL) |
| `020f0000000000000000000000000000.drbin` | Utils |
| `05120000000000000000000000000000.drbin` | Security |
| `020b0000000000000000000000000000.drbin` | CMDQ |
| `05070000000000000000000000000000.drbin` | Goodix fingerprint |
| `070c0000000000000000000000000000.drbin` | IRIS_GPIO / DrTui |
| `40188311faf343488db888ad39496f9a.drbin` | Widevine DRM |
| `5020170115e016302017012521300000.drbin` | DRM HDCP common |

---

### 5. fstab.mt6833 Changes

#### 5.1 Dual Filesystem Entries (NEW)

```diff
  system /system erofs ro wait,slotselect,avb=vbmeta_system,logical,first_stage_mount
+ system /system ext4 ro wait,slotselect,avb=vbmeta_system,logical,first_stage_mount

  system_ext /system_ext erofs ro wait,slotselect,avb=vbmeta_system,logical,first_stage_mount
+ system_ext /system_ext ext4 ro wait,slotselect,avb=vbmeta_system,logical,first_stage_mount

  vendor /vendor erofs ro wait,slotselect,avb,logical,first_stage_mount
+ vendor /vendor ext4 ro wait,slotselect,avb,logical,first_stage_mount

  product /product erofs ro wait,slotselect,avb,logical,first_stage_mount
+ product /product ext4 ro wait,slotselect,avb,logical,first_stage_mount
```

**Critical Insight:** Ext4 fallback entries for all EROFS partitions - kernel will try each in order

#### 5.2 Userdata Encryption Options

```diff
  /dev/block/by-name/userdata /data f2fs noatime,nosuid,nodev,discard,noflush_merge,fsync_mode=nobarrier,reserve_root=134217,resgid=1065,inlinecrypt wait,check,formattable,quota,reservedsize=128m,latemount,resize,readahead_size_kb=512,checkpoint=fs,fileencryption=aes-256-xts:aes-256-cts:v2+inlinecrypt_optimized,keydirectory=/metadata/vold/metadata_encryption,fsverity
```

**Note:** `fsverity` is still present in fstab.mt6833 but removed from recovery.fstab

---

### 6. fstab.emmc Changes

Same dual filesystem entries added.

**Important difference in userdata:**
```diff
- fileencryption=aes-256-xts:aes-256-cts:v2+inlinecrypt_optimized,keydirectory=/metadata/vold/metadata_encryption,fsverity
+ fileencryption=aes-256-xts:aes-256-cts:v2,keydirectory=/metadata/vold/metadata_encryption,fsverity
```

**Changed:** `v2+inlinecrypt_optimized` → `v2`

---

### 7. recovery.fstab Changes

```diff
- /dev/block/by-name/userdata     /data                   f2fs            ... fileencryption=aes-256-xts:aes-256-cts:v2+inlinecrypt_optimized,keydirectory=/metadata/vold/metadata_encryption,fsverity
+ /dev/block/by-name/userdata     /data                   f2fs            ... fileencryption=aes-256-xts:aes-256-cts:v2+inlinecrypt_optimized,keydirectory=/metadata/vold/metadata_encryption
```

**REMOVED:** `fsverity` from userdata mount options

---

### 8. ueventd.rc (NEW FILE)

**Complete device permission configuration:**

```
import /vendor/etc/ueventd.rc
import /odm/etc/ueventd.rc

firmware_directories /etc/firmware/ /odm/firmware/ /vendor/firmware/ /firmware/image/
uevent_socket_rcvbuf_size 16M

# Subsystem definitions
subsystem graphics, drm, input, sound, dma_heap

# Device permissions
/dev/binder               0666   root       root
/dev/hwbinder             0666   root       root
/dev/vndbinder            0666   root       root
/dev/dma_heap/*           0444   system     system
/dev/dri/*                0666   root       graphics
```

**Critical Insight:** Required for proper HAL communication in recovery

---

### 9. Removed Files

| File | Lines Removed | Reason |
|------|---------------|--------|
| `compatibility_matrix.device.xml` | 1416 | VINTF matrix conflicts with recovery |
| `trustonic.rc` | 60 | Merged into `init.tee.rc` |
| `tee.rc` | 40 | Merged into `init.tee.rc` |

---

## 🎯 Detection Patterns for CryptX

### Pattern 1: A15+ Device Detection

```javascript
function detectA15PlusDevice(boardConfig, deviceMk) {
  const indicators = {
    platformVersion: /^99\.\d+\.\d+$/.test(boardConfig.PLATFORM_VERSION),
    noMetadataDecrypt: !boardConfig.TW_INCLUDE_FBE_METADATA_DECRYPT,
    noMetadataPartition: !boardConfig.BOARD_USES_METADATA_PARTITION,
    hasKeymint: deviceMk.packages?.includes('android.hardware.security.keymint'),
    hasSecureclock: deviceMk.packages?.includes('android.hardware.security.secureclock'),
    hasSharedsecret: deviceMk.packages?.includes('android.hardware.security.sharedsecret')
  };
  
  const score = Object.values(indicators).filter(Boolean).length;
  
  return {
    isA15Plus: score >= 3,
    confidence: score / 6,
    indicators: indicators
  };
}
```

### Pattern 2: Trustonic TEE Readiness Check

```javascript
function checkTrustonicReadiness(state) {
  const checks = {
    // Init configuration
    hasInitTeeRc: state.files.some(f => f.path.includes('init.tee.rc')),
    hasMobicoreService: state.initServices.some(s => 
      s.name === 'mobicore' && s.binary.includes('mcDriverDaemon')
    ),
    
    // mcDriverDaemon arguments
    hasSfsReformat: state.initServices.some(s => 
      s.args?.includes('--sfs-reformat')
    ),
    hasP1Arg: state.initServices.some(s => 
      s.args?.includes('--P1 /mnt/vendor/persist/mcRegistry')
    ),
    
    // Keymaster configuration
    hasTrustonicKeymaster: state.initServices.some(s => 
      s.name.includes('keymaster') && s.name.includes('trustonic')
    ),
    keymasterUsesEarlyHal: state.initServices.some(s => 
      s.name.includes('keymaster') && s.class === 'early_hal'
    ),
    
    // Directory structure
    hasMcRegistry: state.fstab.some(e => 
      e.mountPoint.includes('persist') || e.path.includes('mcRegistry')
    ),
    hasUeventdRc: state.files.some(f => f.path.includes('ueventd.rc'))
  };
  
  const score = Object.values(checks).filter(Boolean).length;
  
  return {
    ready: score >= 6,
    score: score,
    maxScore: 8,
    checks: checks,
    missing: Object.entries(checks)
      .filter(([k, v]) => !v)
      .map(([k]) => k)
  };
}
```

### Pattern 3: Dual Filesystem Detection

```javascript
function detectFilesystemFallback(fstabEntries) {
  const mountPoints = {};
  
  fstabEntries.forEach(entry => {
    const key = entry.mountPoint;
    if (!mountPoints[key]) mountPoints[key] = [];
    mountPoints[key].push(entry.fsType);
  });
  
  const results = [];
  
  Object.entries(mountPoints).forEach(([point, types]) => {
    const hasErofs = types.includes('erofs');
    const hasExt4 = types.includes('ext4');
    
    if (hasErofs && hasExt4) {
      results.push({
        mountPoint: point,
        status: 'optimal',
        message: 'Has EROFS with ext4 fallback'
      });
    } else if (hasErofs && !hasExt4) {
      results.push({
        mountPoint: point,
        status: 'warning',
        message: 'EROFS only - consider adding ext4 fallback'
      });
    }
  });
  
  return results;
}
```

### Pattern 4: Keymaster Service Validation

```javascript
function validateKeymasterService(service, teeType) {
  const issues = [];
  const positives = [];
  
  // Check class
  if (service.class === 'early_hal') {
    positives.push('Uses early_hal class (recommended)');
  } else if (service.oneshot) {
    issues.push({
      severity: 'medium',
      message: 'Keymaster configured as oneshot',
      recommendation: 'Consider using class early_hal for better HAL lifecycle'
    });
  }
  
  // Check user
  if (service.user === 'nobody') {
    positives.push('Runs as nobody user (secure)');
  } else if (service.user === 'root') {
    issues.push({
      severity: 'low',
      message: 'Keymaster running as root',
      recommendation: 'Consider using user nobody with proper seclabel'
    });
  }
  
  // Check interface declarations
  if (service.interfaces && service.interfaces.length > 0) {
    positives.push(`Has ${service.interfaces.length} interface declaration(s)`);
  } else {
    issues.push({
      severity: 'medium',
      message: 'Missing interface declarations',
      recommendation: 'Add explicit interface declarations for HAL registration'
    });
  }
  
  // Check SELinux context
  if (service.seclabel === 'u:r:recovery:s0') {
    positives.push('Has recovery SELinux context');
  }
  
  return { issues, positives };
}
```

### Pattern 5: Security Packages Check

```javascript
const REQUIRED_A15_PACKAGES = [
  'android.hardware.security.keymint',
  'android.hardware.security.secureclock',
  'android.hardware.security.sharedsecret'
];

function validateA15SecurityPackages(deviceMk, isA15Plus) {
  if (!isA15Plus) return { status: 'skip', reason: 'Not an A15+ device' };
  
  const packages = deviceMk.packages || [];
  const missing = REQUIRED_A15_PACKAGES.filter(p => !packages.includes(p));
  
  if (missing.length === 0) {
    return {
      status: 'pass',
      message: 'All A15+ security packages present'
    };
  } else {
    return {
      status: 'fail',
      severity: 'high',
      message: 'Missing required A15+ security packages',
      missing: missing,
      recommendation: `Add to device.mk:\nPRODUCT_PACKAGES += \\\n    ${missing.join(' \\\n    ')}`
    };
  }
}
```

---

## 📋 Issue Severity Matrix

| Issue | Severity | Condition |
|-------|----------|-----------|
| Missing keymint packages on A15+ | 🔴 Critical | Platform version 99.x.x |
| Keymaster using oneshot on A15+ | 🟡 Medium | Class not early_hal |
| EROFS without ext4 fallback | 🟡 Medium | For recovery compatibility |
| Missing init.tee.rc for Trustonic | 🟡 Medium | Trustonic TEE detected |
| Missing ueventd.rc | 🟡 Medium | Recovery environment |
| Keymaster running as root | 🟢 Low | Security best practice |
| Missing interface declarations | 🟡 Medium | HAL registration |
| Platform version 99.x.x | ✅ Info | A15+ workaround (not error) |
| Missing FBE_METADATA_DECRYPT | ✅ Info | Valid for A15+ (not error) |

---

## 🔄 Decrypt Flow Update for A15+

```
[Boot]
    ↓
[on fs] → wait UFS device → symlink bootdevice
    ↓
[on early-fs] → start vold
    ↓
[on late-fs] → mount_all fstab.mt6833
    ↓
[on post-fs] → start boot-hal-1-2 → preloader symlinks
    ↓
[on property:crypto.ready=1] → start keymaster (only)
    ↓
[on property:hwservicemanager.ready=true]
    → start mobicore (mcDriverDaemon)
    → start gatekeeper
    → start keymaster
    ↓
[Keymaster registers with hwservicemanager]
    ↓
[TWRP requests decryption via keymaster]
    ↓
[Keymaster communicates with TEE via mcDriverDaemon]
    ↓
[Decryption keys retrieved]
    ↓
[Data partition mounted - DECRYPTED]
```

---

## 🛠️ Recommended Code Additions to CryptX

### 1. TEE_DB Enhancement for Trustonic

```javascript
trustonic: {
  name: 'Trustonic (Kinibi/t-base)',
  color: '#7CACF8',
  icon: 'shield',
  
  // Existing detection patterns
  daemons: ['mcDriverDaemon', 'mobicore'],
  services: ['mcDriverDaemon', 'mobicore', 'tbaseLoader', 'mc_provision', 'TbStorageDaemon'],
  hals: ['trustonic', 'tbase', 'mobicore', 'kinibi'],
  kmBins: ['android.hardware.keymaster@4.0-service.trustonic', 'android.hardware.keymaster@4.1-service.trustonic'],
  gkBins: ['android.hardware.gatekeeper@1.0-service', 'android.hardware.gatekeeper@1.0-service.trustonic'],
  devNodes: ['/dev/mobicore', '/dev/mobicore-user', '/dev/t-base-tui'],
  rcPat: [/mcDriverDaemon/i, /mobicore/i, /tbaseLoader/i, /trustonic/i, /mcRegistry/i],
  
  // NEW: Recovery-specific configuration
  recoveryConfig: {
    daemonArgs: ['--sfs-reformat', '--P1', '/mnt/vendor/persist/mcRegistry'],
    requiredDirs: [
      '/mnt/vendor/persist/mcRegistry',
      '/data/vendor/mcRegistry',
      '/data/vendor/key_provisioning',
      '/data/vendor_de/0/cryptoeng'
    ],
    criticalDrbins: [
      '06090000000000000000000000000000.drbin',  // DRM keyinstall - CRITICAL
      '020f0000000000000000000000000000.drbin',  // Utils
      '05120000000000000000000000000000.drbin',  // Security
      '070c0000000000000000000000000000.drbin',  // IRIS_GPIO/DrTui - often missed!
    ],
    keymasterConfig: {
      class: 'early_hal',
      user: 'nobody',
      interfaces: [
        'android.hardware.keymaster@4.0::IKeymasterDevice',
        'android.hardware.keymaster@4.1::IKeymasterDevice'
      ],
      seclabel: 'u:r:recovery:s0'
    },
    serviceStartOrder: {
      cryptoReady: ['vendor.keymaster-4-1-trustonic'],
      hwservicemanagerReady: ['mobicore', 'vendor.gatekeeper-1-0', 'vendor.keymaster-4-1-trustonic']
    }
  },
  
  desc: 'Samsung/MediaTek Kinibi TEE. Uses mcDriverDaemon with --sfs-reformat flag for recovery.',
  infoLibs: ['libMcClient.so', 'libTeeClient.so', 'libMcRegistry.so'],
}
```

### 2. A15+ Detection Module

```javascript
const A15_COMPAT_MODULE = {
  name: 'Android 15+ Compatibility',
  
  detect(boardConfig) {
    const platformVersion = boardConfig.PLATFORM_VERSION || '';
    return {
      isA15Compat: /^99\.\d+\.\d+$/.test(platformVersion),
      platformVersion: platformVersion
    };
  },
  
  validate(boardConfig, deviceMk) {
    const detection = this.detect(boardConfig);
    const issues = [];
    const positives = [];
    
    if (detection.isA15Compat) {
      positives.push({
        type: 'info',
        message: `Platform version ${detection.platformVersion} indicates A15+ compatibility mode`
      });
      
      // Check for removed metadata flags (should be removed for A15+)
      if (!boardConfig.TW_INCLUDE_FBE_METADATA_DECRYPT && !boardConfig.BOARD_USES_METADATA_PARTITION) {
        positives.push({
          type: 'best_practice',
          message: 'Metadata decrypt flags correctly removed for A15+'
        });
      } else {
        issues.push({
          severity: 'medium',
          message: 'Metadata decrypt flags may cause issues on A15+',
          recommendation: 'Consider removing TW_INCLUDE_FBE_METADATA_DECRYPT and BOARD_USES_METADATA_PARTITION'
        });
      }
      
      // Check for security packages
      const packages = deviceMk.packages || [];
      const missing = [
        'android.hardware.security.keymint',
        'android.hardware.security.secureclock',
        'android.hardware.security.sharedsecret'
      ].filter(p => !packages.includes(p));
      
      if (missing.length > 0) {
        issues.push({
          severity: 'high',
          message: 'Missing A15+ security packages',
          missing: missing
        });
      } else {
        positives.push({
          type: 'pass',
          message: 'All A15+ security packages present'
        });
      }
    }
    
    return { detection, issues, positives };
  }
};
```

### 3. Fstab Dual Filesystem Analyzer

```javascript
function analyzeFstabFallback(fstabEntries) {
  const analysis = {
    partitions: {},
    recommendations: []
  };
  
  // Group by mount point
  fstabEntries.forEach(entry => {
    const point = entry.mountPoint;
    if (!analysis.partitions[point]) {
      analysis.partitions[point] = {
        entries: [],
        fsTypes: []
      };
    }
    analysis.partitions[point].entries.push(entry);
    analysis.partitions[point].fsTypes.push(entry.fsType);
  });
  
  // Analyze each partition
  Object.entries(analysis.partitions).forEach(([point, data]) => {
    const types = [...new Set(data.fsTypes)];
    
    if (types.includes('erofs')) {
      if (types.includes('ext4')) {
        analysis.partitions[point].status = 'optimal';
        analysis.partitions[point].message = 'EROFS with ext4 fallback configured';
      } else {
        analysis.partitions[point].status = 'warning';
        analysis.partitions[point].message = 'EROFS only - no fallback';
        analysis.recommendations.push({
          partition: point,
          message: `Consider adding ext4 fallback for ${point}`,
          example: `${point} ${point} ext4 ro wait,slotselect,avb,logical,first_stage_mount`
        });
      }
    }
  });
  
  return analysis;
}
```

---

## 📊 Summary Table

| Category | Old Behavior | New Behavior (A15+) |
|----------|--------------|---------------------|
| Platform Version | 14 | 99.87.36 (workaround) |
| Metadata Decrypt | Enabled | Disabled |
| Security Packages | DRM only | Keymint + Secureclock + Sharedsecret |
| Keymaster Class | oneshot | early_hal |
| Keymaster User | root | nobody |
| Filesystem | EROFS only | EROFS + ext4 fallback |
| TEE Config Files | tee.rc + trustonic.rc | init.tee.rc (merged) |
| Service Start Order | crypto.ready starts all | crypto.ready starts keymaster only |
| USB Config | MTP + ADB | ADB only |
| mtk_plpath_utils | /vendor/bin/ | /system/bin/ |

---

## ✅ Test Cases for CryptX

1. **Tecno LH8n (this commit)** - Should pass all checks
2. **A15+ device with platform 99.x.x** - Should recognize as compat mode
3. **Trustonic device without init.tee.rc** - Should warn
4. **EROFS-only fstab** - Should suggest fallback
5. **Keymaster with oneshot** - Should suggest early_hal

---

*Review v2.0 - Complete analysis of commit 97f07ab*
*All 11 files analyzed with before/after comparison*
