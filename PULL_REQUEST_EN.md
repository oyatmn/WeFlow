# fix(export): "Date Range" file naming mode not working in export global config

## Problem

In the **Export page → Global Settings → Export File Naming Mode**, selecting "Date Range Mode" (`date-range`) has no effect. The exported files always use the "Classic Mode" (`classic`) naming format without any date range information. The option is completely ignored.

## Root Cause

The `fileNamingMode` config value is lost at 4 breakpoints along the data flow from saving (`ExportDefaultsSettingsForm` → `configService.setExportDefaultFileNamingMode()`) to actual usage (Electron side `ExportOrchestrator` → `normalizeFileNamingMode()`):

### 1. `types.ts` — `ExportOptions` interface missing `fileNamingMode` field

The frontend type definition lacks this property, so TypeScript cannot track its existence. The data is already lost at the very first step of data flow.

### 2. `constants.ts` — `createDefaultExportOptions()` missing default value

No default value is set when creating export options, leaving `fileNamingMode` as `undefined`.

### 3. `useExportConfig.ts` — Not loaded on init, not persisted in `updateOptions`

- Initial load does not call `getExportDefaultFileNamingMode()`
- Changes via `updateOptions` are not persisted to storage via `setExportDefaultFileNamingMode()`

### 4. `ExportPage.tsx` — `handleExportDefaultsChanged` drops the `fileNamingMode` patch

The callback fired when closing the global settings modal only passes `format` and `avatars` to `updateOptions`, completely ignoring `fileNamingMode`.

## Fix

Bridged each gap in the data flow so the config value flows end-to-end: save → load → propagate → use.

## Changes

| File | Change Type | Description |
|------|-------------|-------------|
| `src/pages/Export/types.ts` | New field | Added `fileNamingMode: 'classic' \| 'date-range'` to `ExportOptions` interface |
| `src/pages/Export/constants.ts` | New default | Added `fileNamingMode: 'classic'` default in `createDefaultExportOptions()` |
| `src/pages/Export/hooks/useExportConfig.ts` | Load + persist | Added `getExportDefaultFileNamingMode()` to initial `Promise.all`; assign loaded value to `newOptions.fileNamingMode`; added `setExportDefaultFileNamingMode()` persistence in `updateOptions` |
| `src/pages/Export/ExportPage.tsx` | Propagate + fix key | Propagate `fileNamingMode` in `handleExportDefaultsChanged`; fixed incorrect key name `defaultFormat` → `format` |

## Verification

- Tested on Windows desktop client: selecting "Date Range Mode" produces correct filenames with date range tokens (e.g. `Private_ZhangSan_20250101-20250331.json`)
- TypeScript type check passes, build succeeds with no errors
