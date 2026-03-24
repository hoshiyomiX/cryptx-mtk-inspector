# CryptX MTK Decrypt Inspector - Work Log

## Project Info
- **Repository:** https://github.com/hoshiyomiX/cryptx-mtk-inspector
- **Live Site:** https://hoshiyomiX.github.io/cryptx-mtk-inspector
- **Current Version:** v3.2

## Work History

### 2024-03-24 - v3.2 Release
**Task:** Fix Analyze button and update to v3.2

**Changes Made:**
1. Added version-specific constants:
   - `VERSION_INIT_SERVICES` - Init service requirements per Android version
   - `VERSION_VINTF_MANIFESTS` - VINTF manifest requirements
   - `VERSION_KERNEL_MODULES` - Kernel module requirements
   - `CRITICAL_DRBINS` - Critical Trustonic drbin files

2. Added Android version selector dropdown (Auto/11-15)

3. Enhanced VINTF manifest validation:
   - Keymaster/gatekeeper detection
   - Version-specific VINTF requirements
   - A15+ keymint/secureclock/sharedsecret checks

4. Added version-specific init service checks

5. Added kernel module requirement checks

6. Updated version label to v3.2

---
