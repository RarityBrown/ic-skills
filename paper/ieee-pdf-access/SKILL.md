---
name: ieee-pdf-access
description: Download IEEE Xplore article PDFs by DOI, arnumber, or document URL using institution/IP access or a user-provided browser session. Use for conservative serial IEEE PDF retrieval with network preflight, metadata extraction, stampPDF/ielx fallback, and %PDF verification; not for broad literature search, paywall bypass, or automated university SSO.
---

# IEEE PDF Access

Download a real IEEE Xplore PDF for a known paper. Keep the workflow narrow, serial, and conservative.

Do not parallelize IEEE downloads. Do not bypass paywalls. Do not ask for or store university passwords.

## Core Rule

A download succeeds only if the saved response starts with:

```text
%PDF
```

Do not trust HTTP 200, `.pdf` filenames, or `Content-Type` alone.

Reject HTML/script responses such as:

```text
<!DOCTYPE html>
<script>
<APM_DO_NOT_TOUCH>
```

## Network Permission

IEEE PDF retrieval requires live network access from the local shell. Request one scoped network escalation for the serial download flow instead of repeatedly trying sandboxed network commands. If network access is already enabled, run normally without escalation.

## Fast Path Checklist

1. Resolve input to IEEE `arnumber`.
2. Run access preflight.
3. Visit `https://ieeexplore.ieee.org/document/<arnumber>` with a cookie jar/session.
4. Extract `xplGlobal.document.metadata`.
5. Build PDF candidates:
   - derived `stampPDF`
   - derived `/ielxN/...pdf`
   - original `pdfUrl`
   - original `pdfPath`
6. Try candidates serially with `Referer` set to the document page.
7. Sleep 2-4 seconds between attempts.
8. Stop at the first `%PDF` response and report the method used.

## Inputs

Accept:

- IEEE arnumber: `8662406`
- IEEE document URL: `https://ieeexplore.ieee.org/document/8662406`
- DOI: `10.1109/ISSCC.2019.8662406`
- Exact title, only when DOI/arnumber is unavailable

For title input, use it only to resolve an arnumber. Confirm the resolved title/DOI before downloading.

## Access Preflight

Before downloading, check whether current access plausibly has IEEE entitlement.

Use available signals:

- Local network/DNS: `ipconfig /all`, DNS suffix, search suffix, Wi-Fi/network name
- IEEE document page fields: `fullTextAccess`, `isAccessFromInstitution`, `institutionName`
- Page text such as `Access provided by ...`

Treat preflight as passing if evidence shows a university/institution network or IEEE access, for example:

```text
Stanford
Tsinghua
Oxford
```

Permission-denied local commands do not mean preflight failed. Fall back to other evidence.

If preflight does not pass:

1. Make only one lightweight attempt:
   - visit document page with cookie jar
   - request derived `stampPDF` once
   - verify `%PDF`
2. If it fails, stop and tell the user the current network/session likely lacks IEEE authorization.

## Resolve arnumber

Use these patterns:

```text
/document/<number>
arnumber=<number>
```

For DOI, resolve through DOI/Crossref/OpenAlex/IEEE metadata until an IEEE document URL or arnumber is found.

## Document Metadata

Fetch:

```text
https://ieeexplore.ieee.org/document/<arnumber>
```

Use:

- browser-like `User-Agent`
- cookie jar or user-provided session cookies
- serial request

Extract `xplGlobal.document.metadata` when available.

Useful fields:

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

## PDF Candidates

Do not use `pdfUrl` / `pdfPath` blindly. IEEE often returns wrappers or HTML.

Try in this order.

### 1. Derived stampPDF

```text
https://ieeexplore.ieee.org/stampPDF/getPDF.jsp?tp=&arnumber=<arnumber>
```

### 2. Derived ielx path

If metadata has:

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

### 3. Original pdfUrl

Use only as fallback. It may be a wrapper such as:

```text
/stamp/stamp.jsp?tp=&arnumber=<arnumber>
```

### 4. Original pdfPath

Use only as fallback. `/ielN/...pdf` may return HTML even when `/ielxN/...pdf` succeeds.

## Minimal Curl Template

Use this shape for the simplest HTTP/cookie-jar path. Keep requests serial. Always quote IEEE URLs because query strings contain `&`.

```bash
ARNUMBER="8662406"
DOC_URL="https://ieeexplore.ieee.org/document/${ARNUMBER}"
JAR="ieee-cookies.txt"
OUT="${ARNUMBER}.pdf"

curl -L -c "$JAR" -b "$JAR" \
  -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0 Safari/537.36" \
  -H "Accept: text/html,*/*" \
  "$DOC_URL" \
  -o "${ARNUMBER}.html"

curl -L -c "$JAR" -b "$JAR" \
  -A "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/125.0 Safari/537.36" \
  -H "Accept: application/pdf,application/octet-stream;q=0.9,*/*;q=0.8" \
  -H "Referer: ${DOC_URL}" \
  "https://ieeexplore.ieee.org/stampPDF/getPDF.jsp?tp=&arnumber=${ARNUMBER}" \
  -o "$OUT"

head -c 4 "$OUT"
```

Success requires:

```text
%PDF
```

If the first bytes are not `%PDF`, delete or ignore the file and try the next candidate URL, such as the derived `/ielxN/...pdf` path.

## Rate Limits

- Use one paper at a time unless explicitly asked otherwise.
- Batch downloads must be strictly serial.
- Sleep 3-5 seconds between candidate attempts or papers.
- If IEEE returns rate-limit, bot-check, repeated HTML errors, or 418/502 responses, stop or back off. Do not increase concurrency.

## Stop Conditions

Stop and report failure when:

- preflight fails and the single lightweight `stampPDF` attempt is not `%PDF`
- document page cannot be loaded
- metadata cannot be extracted and direct `stampPDF` also fails
- all candidates return non-PDF responses
- IEEE redirects to login/access-denied pages
- repeated anti-bot/rate-limit errors appear

## User Report

Report briefly:

```text
Network/access preflight: passed/failed/uncertain
Resolved paper: <title>, arnumber=<arnumber>
PDF result: success/failure
Successful method: stampPDF / ielx / other
Verification: first bytes are %PDF
Saved file: <path>
Failure reason: <short likely cause>
```

Do not expose cookies, tokens, storage state contents, or signed URLs.
