---
name: ieee-pdf-access
description: Download IEEE Xplore article PDFs by arnumber, DOI, or document URL using institution/IP access or a user-provided browser session. Use for conservative serial IEEE PDF retrieval with a direct stampPDF fast path, cookie-jar session reuse, fallback metadata/ielx handling, and %PDF verification; not for broad literature search, paywall bypass, or automated university SSO.
---

# IEEE PDF Access

Download a real IEEE Xplore PDF for a known paper. Keep the workflow narrow, serial, and conservative.

Do not parallelize IEEE downloads. Do not bypass paywalls. Do not ask for or store university passwords.

## Core Rule

A download succeeds only if the saved response starts with:

```text
%PDF
```

Do not trust HTTP 200, `.pdf` filenames, or `Content-Type` alone. Reject HTML/script responses such as `<!DOCTYPE html>`, `<script>`, or `<APM_DO_NOT_TOUCH>`.

## Network Permission

IEEE PDF retrieval requires live network access from the local shell. Request one scoped network escalation for the serial download flow instead of repeatedly trying sandboxed network commands. If network access is already enabled, run normally without escalation.

## Inputs

Accept:

- IEEE arnumber: `8662406`
- IEEE document URL: `https://ieeexplore.ieee.org/document/8662406`
- DOI: `10.1109/ISSCC.2019.8662406`
- Exact title, only when DOI/arnumber is unavailable

For title or DOI input, resolve to an IEEE `arnumber` before downloading. Confirm the resolved title/DOI if there is ambiguity.

## Main Flow

1. Resolve input to an IEEE `arnumber`.
2. If `arnumber` is known, try the direct `stampPDF` fast path first.
3. Verify the saved bytes start with `%PDF`.
4. If fast path fails, fall back to document-page access, metadata extraction, and candidate URLs.
5. Stop at the first `%PDF` response and report the method used.

## Direct StampPDF Fast Path

For a known `arnumber`, first try a single direct `stampPDF` request. Do not fetch the document page or metadata first.

Use:

- browser-like `User-Agent`
- `Referer: https://ieeexplore.ieee.org/document/<arnumber>`
- `-L` / redirect following
- one temporary cookie jar reused by `-c` and `-b`
- quoted IEEE URLs, because query strings contain `&`

IEEE may set access/session cookies during the redirect to `tag=1`. The cookie names can change; do not depend on specific names. Reuse the same temporary cookie jar throughout the run and delete it after success or final failure unless the user asks to preserve the session.

### Minimal Curl Template

```bash
ARNUMBER="8662406"
DOC_URL="https://ieeexplore.ieee.org/document/${ARNUMBER}"
PDF_URL="https://ieeexplore.ieee.org/stampPDF/getPDF.jsp?tp=&arnumber=${ARNUMBER}"
JAR="ieee-cookies.txt"
OUT="${ARNUMBER}.pdf"

curl -L -c "$JAR" -b "$JAR" \
  -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0 Safari/537.36" \
  -H "Accept: application/pdf,application/octet-stream;q=0.9,*/*;q=0.8" \
  -H "Referer: ${DOC_URL}" \
  "$PDF_URL" \
  -o "$OUT"

head -c 4 "$OUT"
```

Success requires:

```text
%PDF
```

For PowerShell, always quote the PDF URL:

```powershell
$arnumber = "8662406"
$docUrl = "https://ieeexplore.ieee.org/document/$arnumber"
$pdfUrl = "https://ieeexplore.ieee.org/stampPDF/getPDF.jsp?tp=&arnumber=$arnumber"
$jar = "ieee-cookies.txt"
$out = "$arnumber.pdf"

curl.exe -L -c $jar -b $jar `
  -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0 Safari/537.36" `
  -H "Accept: application/pdf,application/octet-stream;q=0.9,*/*;q=0.8" `
  -H "Referer: $docUrl" `
  -o $out `
  "$pdfUrl"

Get-Content -Encoding Byte -TotalCount 4 $out
```

If the first bytes are not `%PDF`, delete or ignore the file and continue to fallback.

## Resolve arnumber

Use these patterns:

```text
/document/<number>
arnumber=<number>
```

For DOI, resolve through DOI/Crossref/OpenAlex/IEEE metadata until an IEEE document URL or arnumber is found. For exact title input, use search only to resolve an arnumber, not for broad literature discovery.

## Fallback: Access And Metadata

Use fallback only when direct `stampPDF` is not `%PDF`, when resolving DOI/title requires the document page, or when the user asks for access diagnostics.

Fetch:

```text
https://ieeexplore.ieee.org/document/<arnumber>
```

Use the same browser-like `User-Agent`, a cookie jar or user-provided session cookies, and serial requests.

Extract `xplGlobal.document.metadata` when available. Useful fields:

```text
displayDocTitle
title
doi
articleNumber
pdfUrl
pdfPath
fullTextAccess
isAccessFromInstitution
isOpenAccess
isFreeDocument
institutionName
```

Access clues can include local network/DNS (`ipconfig /all`, DNS suffix, Wi-Fi/network name), page fields such as `fullTextAccess` and `isAccessFromInstitution`, or page text such as `Access provided by ...`. Permission-denied local commands do not prove access failure; rely on download verification.

If access appears absent, make only one lightweight `stampPDF` attempt and stop if it is not `%PDF`. Tell the user the current network/session likely lacks IEEE authorization.

## Fallback PDF Candidates

Do not use metadata `pdfUrl` / `pdfPath` blindly. IEEE often returns wrappers or HTML.

Try candidates serially with the document URL as `Referer`:

1. Derived `stampPDF`:

```text
https://ieeexplore.ieee.org/stampPDF/getPDF.jsp?tp=&arnumber=<arnumber>
```

2. Derived `ielx` path. If metadata has:

```text
pdfPath=/iel7/8656625/8662285/08662406.pdf
```

try:

```text
https://ieeexplore.ieee.org/ielx7/8656625/8662285/08662406.pdf
```

General rule:

```text
/ielN/... -> /ielxN/...
```

3. Original `pdfUrl`, only as fallback. It may be a wrapper such as `/stamp/stamp.jsp?tp=&arnumber=<arnumber>`.
4. Original `pdfPath`, only as fallback. `/ielN/...pdf` may return HTML even when `/ielxN/...pdf` succeeds.

## Rate Limits

- Use one paper at a time unless explicitly asked otherwise.
- Batch downloads must be strictly serial.
- Sleep 2-5 seconds between fallback candidate attempts or papers.
- If IEEE returns rate-limit, bot-check, repeated HTML errors, or 418/502 responses, stop or back off. Do not increase concurrency.

## Stop Conditions

Stop and report failure when:

- direct `stampPDF` and fallback candidates all return non-PDF responses
- preflight/access clues suggest no entitlement and the single lightweight `stampPDF` attempt is not `%PDF`
- document page cannot be loaded when fallback requires it
- metadata cannot be extracted and direct `stampPDF` also fails
- IEEE redirects to login/access-denied pages
- repeated anti-bot/rate-limit errors appear

## User Report

Report briefly:

```text
Resolved paper: <title if known>, arnumber=<arnumber>
Network/access: passed/failed/uncertain if checked
PDF result: success/failure
Successful method: direct stampPDF / ielx / other
Verification: first bytes are %PDF
Saved file: <path>
Failure reason: <short likely cause>
```

Do not expose cookies, tokens, storage state contents, or signed URLs.
