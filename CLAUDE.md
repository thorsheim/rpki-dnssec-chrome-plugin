# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is an RPKI (Resource Public Key Infrastructure) Chrome browser extension that checks if websites are protected by RPKI. The extension provides real-time RPKI protection status and network ownership information for any website.

## Architecture

The extension consists of:

- `manifest.json` - Chrome extension manifest v3 with required permissions and strict CSP
- `popup.html` - Extension popup UI with status display (no inline styles)
- `popup.css` - External stylesheet for popup UI
- `popup.js` - Frontend logic and UI updates
- `background.js` - Service worker handling API calls and data processing
- `README.md` - Installation and usage documentation

## Key Features

- **RPKI Protection Check**: Verifies if website IPs are in Cloudflare's RPKI records
- **IPv6 Support**: Displays both IPv4 and IPv6 addresses when available
- **Network Information**: Shows AS number, organization, and location for all sites
- **DNS Resolution**: Uses Google DNS-over-HTTPS for both A and AAAA record lookup
- **Advanced Caching**: 12-hour default cache with configurable timeout (1-96 hours) and manual flush option
- **TLS Analysis**: Displays TLS version, cipher suites, and security level information
- **Post-Quantum Cryptography Detection**: Highlights PQC algorithms in TLS connections
- **Security Reporting**: Email reporting for sites missing RPKI and/or DNSSEC protection
- **Color-coded Status**: Green "PROTECTED", Red "NOT PROTECTED", Purple "ERROR"

## APIs Used

- `https://rpki.cloudflare.com/rpki.json` - RPKI ROA data
- `https://dns.google/resolve` - DNS-over-HTTPS resolution
- `https://ipinfo.io/{ip}/json` - AS and geolocation information

## Development

### Testing the Extension

1. Load unpacked extension in `chrome://extensions/` (enable Developer mode)
2. Click extension icon on any website to test functionality
3. Check browser console for background script logs
4. Use Developer Tools on popup for frontend debugging

### Key Functions

#### Background Service Worker (background.js)
- `fetchRPKIData()` - Downloads and caches RPKI data with rate limiting
- `checkRPKIProtection()` - Performs CIDR prefix matching with ROA validation
- `validateROA()` - Validates ROA structure (prefix, ASN, maxLength)
- `resolveHostname()` - Resolves both IPv4 and IPv6 addresses
- `resolveIPv4()` - Queries A records for IPv4 addresses with rate limiting
- `resolveIPv6()` - Queries AAAA records for IPv6 addresses with rate limiting
- `validateIPv4()` - Validates IPv4 address format
- `validateIPv6()` - Validates IPv6 address format (ReDoS-safe regex)
- `validateHostname()` - Validates and sanitizes hostnames with IDN/punycode support
- `checkDNSSECStatus()` - Checks DNSSEC with record data validation
- `getASInfo()` - Gets network ownership data for IPv4/IPv6 with rate limiting
- `isIpInPrefix()` - CIDR matching with integer overflow protection
- `handleFlushCache()` - Clears RPKI data cache
- `handleSetCacheTimeout()` - Sets custom cache timeout
- `handleGetCacheSettings()` - Retrieves cache status and settings
- `RateLimiter` class - Rate limits API calls to prevent abuse

#### Popup UI (popup.js)
- `updateIPAddresses()` - Updates both IPv4 and IPv6 display
- `showNetworkInfo()` - Displays AS and location information
- `showDNSSECInfo()` - Displays DNSSEC validation status
- `initializeSettings()` - Sets up cache management UI
- `showReportSection()` - Shows reporting UI for missing protections
- `generateSecurityReport()` - Creates security report with site details

### Permissions Required

- `activeTab` - Access current tab URL
- `storage` - Cache RPKI data
- Host permissions for API endpoints:
  - `https://rpki.cloudflare.com/*` - RPKI data
  - `https://dns.google/*` - DNS-over-HTTPS
  - `https://ipinfo.io/*` - AS information

## RPKI Implementation Details

- Downloads ROA (Route Origin Authorization) list from Cloudflare
- Validates ROA structure before processing (prefix format, ASN format, maxLength range)
- Performs CIDR subnet matching using bitwise operations with overflow protection
- Handles both IPv4 prefixes and variable prefix lengths (0-32)
- Caches RPKI data with configurable timeout (1-96 hours, default 12 hours)
- Timestamp validation prevents cache poisoning attacks
- Provides cache management with manual flush and status monitoring
- Currently RPKI validation supports IPv4 only (IPv6 RPKI support limited in current datasets)

## UI Components

- IP Addresses section: Displays both IPv4 and IPv6 addresses when available
- Network Information section (always shown): AS number, organization, location
- RPKI Details section (protected sites only): IP prefix, RPKI ASN, max length
- DNSSEC Details section: Shows signing and authentication status with validated record data
- Status section with color-coded text indicators
- Report section (missing protections only): Clipboard reporting functionality for security issues
- Settings section: Cache management controls with timeout configuration and flush option
- All styling via external CSS (popup.css) - no inline styles

## Security Features

### Input Validation
- **IPv6 Validation**: ReDoS-safe regex prevents catastrophic backtracking attacks
- **IPv4 Validation**: Strict format checking with range validation
- **Hostname Validation**: IDN/punycode detection and validation, homograph attack warnings
- **ROA Validation**: Validates prefix format, ASN format, and maxLength ranges
- **DNS Response Validation**: Type-checks all DNS record fields before processing
- **DNSSEC Record Validation**: Validates record structure, data type, and size limits

### Attack Prevention
- **Rate Limiting**: Prevents API abuse with configurable limits per endpoint
  - RPKI: 10 requests/minute
  - DNS: 30 requests/minute
  - IPInfo: 20 requests/minute
- **Integer Overflow Protection**: CIDR prefix length validation (0-32 range)
- **Cache Poisoning Prevention**: Timestamp validation rejects future/negative values
- **Information Disclosure**: Generic error messages hide internal details

### Content Security Policy
- Strict CSP with no `'unsafe-inline'` for scripts or styles
- External stylesheets only
- Limited connect-src to required API endpoints
- No eval or dynamic code execution

## Error Handling

- Graceful degradation when APIs are unavailable
- Rate limiting with clear error messages
- Fallback to basic functionality if network info lookup fails
- User-friendly generic error messages prevent information disclosure
- All errors logged internally without exposing sensitive details