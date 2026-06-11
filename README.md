# Horde MAPI

**MAPI utility library** – helper library for Microsoft MAPI data structures in the Horde Framework.

| | |
|---|---|
| **Package** | `horde/mapi` |
| **Upstream** | [github.com/horde/mapi](https://github.com/horde/mapi) |
| **License** | LGPL-2.1 |
| **Author** | Michael J Rubinsky |
| **PHP** | ^8.1 |
| **Dependencies** | `horde/date`, `horde/exception`, PHP extension `bcmath` |

## What is MAPI?

**MAPI** (Messaging Application Programming Interface) is Microsoft's API for messaging, calendars, and contacts—primarily in Exchange and Outlook. In Horde, this library is **not** a full MAPI client or Exchange connector. It provides **binary format helpers** used by Microsoft protocols, especially **Exchange ActiveSync**.

Typical use cases in Horde:

- Encode/decode timezones as MAPI `TIME_ZONE_INFORMATION` blobs (ActiveSync)
- Convert **Global Object Identifier (GOID)** ↔ iCalendar UID (appointments, meeting requests)
- Convert **Windows FILETIME** to Unix timestamps

## Architecture

The library is intentionally small and consists of three classes:

```
lib/Horde/
├── Mapi.php              → Horde_Mapi (static helpers)
└── Mapi/
    ├── Timezone.php      → Horde_Mapi_Timezone
    └── Exception.php     → Horde_Mapi_Exception
```

### `Horde_Mapi`

General MAPI helper functions (all static):

| Method | Description |
|--------|-------------|
| `isLittleEndian()` | Checks the system's byte order |
| `chbo($num)` | Reverses the byte order of an integer (for big-endian systems) |
| `getUidFromGoid($goid)` | Extracts a UID from a Base64-encoded GOID |
| `createGoid($uid)` | Creates a Base64-encoded GOID from a UID |
| `filetimeToUnixtime($ft)` | Converts binary Windows FILETIME to a Unix timestamp |

MAPI structures are encoded in **little-endian** order. On big-endian systems, values are adjusted with `chbo()` before packing/unpacking.

`filetimeToUnixtime()` requires the PHP **bcmath** extension because FILETIME uses 64-bit values that PHP cannot represent natively. Without `bcmath`, it returns `-1`.

### `Horde_Mapi_Timezone`

Converts between Microsoft's timezone blob and PHP/IANA timezones.

Microsoft defines timezones as a binary `TIME_ZONE_INFORMATION` structure (bias, standard/daylight names, `SYSTEMTIME` transitions, etc.). ActiveSync transmits this structure **Base64-encoded** in the WBXML protocol.

| Method | Direction | Description |
|--------|-----------|-------------|
| `getOffsetsFromSyncTZ($data)` | MAPI blob → array | Decodes an ActiveSync timezone blob into an offset hash |
| `getSyncTZFromOffsets($offsets)` | array → MAPI blob | Builds an ActiveSync-compatible timezone blob |
| `getOffsetsFromDate(Horde_Date $date)` | IANA TZ → array | Derives offsets from a `Horde_Date`/timezone |
| `getTimezone($offsets, $expectedTimezone)` | MAPI blob/array → IANA | Guesses the matching Olson timezone |
| `getListOfTimezones($offsets, $expectedTimezone)` | MAPI blob/array → array | All matching IANA identifiers |

Reverse mapping (MAPI offsets → IANA timezone) is **ambiguous**: multiple timezones can share the same offsets. You can pass `$expectedTimezone` to prefer a specific match; otherwise `Horde_Date::getTimezoneAlias()` picks an alias.

Logic for detecting the nth weekday occurrence in a month (e.g. "second Sunday in March") is partly inspired by the [Tine 2.0](http://tine20.org) project.

### `Horde_Mapi_Exception`

Base exception class (extends `Horde_Exception_Wrapped`), e.g. for missing `bcmath` extension or no matching timezone.

## How it works

### 1. Timezone blob (ActiveSync)

```
Horde_Date / IANA timezone
        │
        ▼
getOffsetsFromDate()          ← reads DST/STD transitions via DateTimeZone
        │
        ▼
getSyncTZFromOffsets()        ← pack() into TIME_ZONE_INFORMATION, base64_encode()
        │
        ▼
ActiveSync WBXML (Appointment.timezone)
```

Reverse direction when receiving:

```
Base64 timezone blob from client
        │
        ▼
getOffsetsFromSyncTZ()        ← base64_decode(), unpack()
        │
        ▼
getTimezone() / getListOfTimezones()   ← match against all IANA zones
        │
        ▼
e.g. "Europe/Berlin"
```

### 2. Global Object Identifier (GOID)

GOIDs uniquely identify appointments across Exchange/Outlook. The library supports two formats:

- **Outlook style**: hex representation of the entire GOID, bytes 17–20 set to zero
- **vCal style**: `vCal-Uid` marker in the blob, UID as plain text after it

```
iCalendar UID  ←→  createGoid() / getUidFromGoid()  ←→  Base64 GOID
```

### 3. FILETIME

Windows stores timestamps as 64-bit counters in 100-ns ticks since 1 Jan 1601 UTC. Conversion steps:

1. Hex string from binary data
2. Flip endianness (`_flipEndian`)
3. Hex → bcmath integer (`_hexToBcint`)
4. Subtract epoch offset and divide by 10,000,000 (`_win64ToUnix`)

## Usage in a Horde installation

`horde/mapi` is used mainly indirectly via **ActiveSync**:

| Module | File | Usage |
|--------|------|-------|
| **ActiveSync** | `Horde/ActiveSync/Message/Appointment.php` | `setTimezone()` / `getTimezone()` for appointment sync |
| **ActiveSync** | `Horde/ActiveSync/Message/MeetingRequest.php` | `createGoid()` for meeting UIDs, timezone parsing |
| **Kronolith** | `lib/Event.php` | `_handleEas16Exception()` – resolve timezone from ActiveSync message |

These functions previously lived in `Horde_ActiveSync_Utils` and `Horde_ActiveSync_Timezone`; since Horde 6 they were moved into `horde/mapi` (the old classes are marked `@deprecated`).

## Examples

### Create a timezone blob

```php
$date = new Horde_Date(time(), 'Europe/Berlin');
$offsets = Horde_Mapi_Timezone::getOffsetsFromDate($date);
$blob = Horde_Mapi_Timezone::getSyncTZFromOffsets($offsets);
// $blob → Base64 string for ActiveSync
```

### Parse a timezone blob

```php
$parser = new Horde_Mapi_Timezone();
$tz = $parser->getTimezone($blob, 'Europe/Berlin');
// $tz → e.g. "Europe/Berlin"
```

### GOID ↔ UID

```php
$goid = Horde_Mapi::createGoid('{81412D3C-2A24-4E9D-B20E-11F7BBE92799}');
$uid  = Horde_Mapi::getUidFromGoid($goid);
```

### Convert FILETIME

```php
$binaryFiletime = /* from MAPI pTypDate property */;
$unix = Horde_Mapi::filetimeToUnixtime($binaryFiletime);
```

## Installation

Via Composer (in Horde, typically as a framework dependency):

```bash
composer require horde/mapi
```

In this installation: `dev-FRAMEWORK_6_0` (Horde Framework 6).

## Tests

```bash
cd vendor/horde/mapi
vendor/bin/phpunit
```

Tests live under `test/Unnamespaced/` (`MapiTest.php`, `TimezoneTest.php`) with fixtures for GOIDs and FILETIME.

## Further reading

- [MSDN: TIME_ZONE_INFORMATION](https://learn.microsoft.com/en-us/windows/win32/api/timezoneapi/ns-timezoneapi-time_zone_information)
- [MSDN: Global Object IDs](https://learn.microsoft.com/en-us/previous-versions/office/developer/exchange-server-2010/ee157690(v=exchg.80))
- [Horde Libraries: Horde_Mapi](https://www.horde.org/libraries/Horde_Mapi)
