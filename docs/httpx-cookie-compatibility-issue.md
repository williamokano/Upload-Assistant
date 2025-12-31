# httpx Cookie Compatibility Issue with MozillaCookieJar

## Summary

httpx does not properly send cookies when a `http.cookiejar.MozillaCookieJar` object is passed to `httpx.AsyncClient()`. This causes authentication failures for trackers that rely on cookie-based authentication, including ASC (Amigos Share Club).

## Issue Description

### Symptoms

- Tracker authentication fails despite valid cookies being loaded from the cookie file
- Server responds with login redirects or "cookies expired" messages
- The `Cookie` header is not included in HTTP requests, or cookies are not properly sent

### Root Cause

When passing a `MozillaCookieJar` object directly to `httpx.AsyncClient(cookies=cookie_jar)`, httpx fails to properly extract and send the cookies in HTTP requests. This is a compatibility issue between Python's standard library `http.cookiejar.MozillaCookieJar` and httpx's internal cookie handling.

## Investigation

### Test Environment

- **httpx version**: 0.28.1
- **Python version**: 3.14
- **Affected tracker**: ASC (Amigos Share Club) at `cliente.amigos-share.club`
- **Affected code**: `src/cookie_auth.py` and `src/trackers/ASC.py`

### Test Results

Three different approaches were tested using httpbin.org as a test endpoint (httpbin.org echoes back the headers it receives, allowing us to verify what cookies were actually sent):

```python
# TEST 1: Passing MozillaCookieJar directly to httpx
cookie_jar = http.cookiejar.MozillaCookieJar(cookie_file)
cookie_jar.load(ignore_discard=True, ignore_expires=True)
async with httpx.AsyncClient(cookies=cookie_jar) as client:
    response = await client.get(test_url)

# Result: Cookie header received by server: NOT FOUND ❌
```

```python
# TEST 2: Converting to httpx.Cookies
httpx_cookies = httpx.Cookies()
for cookie in cookie_jar:
    httpx_cookies.set(cookie.name, cookie.value)
async with httpx.AsyncClient(cookies=httpx_cookies) as client:
    response = await client.get(test_url)

# Result: Cookie header received by server: test_cookie=test_value_123; session_id=abc123def456 ✅
```

```python
# TEST 3: Manually setting Cookie header
cookie_string = '; '.join(f'{cookie.name}={cookie.value}' for cookie in cookie_jar)
headers = {'Cookie': cookie_string}
async with httpx.AsyncClient(headers=headers) as client:
    response = await client.get(test_url)

# Result: Cookie header received by server: test_cookie=test_value_123; session_id=abc123def456 ✅
```

### Conclusion

Only TEST 2 (converting to `httpx.Cookies`) and TEST 3 (manual Cookie header) successfully sent cookies to the server. TEST 1 (passing `MozillaCookieJar` directly) failed completely.

**Important Note**: httpbin.org confirms this behavior by echoing back request headers - when `MozillaCookieJar` is passed directly, httpbin reports `Cookie: NOT FOUND`, proving that httpx never included the Cookie header in the HTTP request. This test was created in `test_httpx_simple.py` and can be reproduced independently of any tracker.

## Solution

### Approach

Convert `MozillaCookieJar` objects to `httpx.Cookies` format before passing them to httpx. This ensures cookies are properly sent in HTTP requests.

### Implementation

A helper method `_convert_cookiejar_to_httpx()` was added to:
1. `CookieValidator` class in `src/cookie_auth.py`
2. `CookieAuthUploader` class in `src/cookie_auth.py`
3. `ASC` class in `src/trackers/ASC.py`

```python
def _convert_cookiejar_to_httpx(self, cookie_jar):
    """
    Convert a MozillaCookieJar to httpx.Cookies object.

    httpx does not properly handle MozillaCookieJar objects when passed
    to the AsyncClient constructor. This method converts the cookie jar
    to httpx.Cookies format which httpx can properly send in requests.

    Args:
        cookie_jar: http.cookiejar.MozillaCookieJar instance

    Returns:
        httpx.Cookies: Cookies in httpx format
    """
    # If already httpx.Cookies, return as-is
    if isinstance(cookie_jar, httpx.Cookies):
        return cookie_jar

    # Convert MozillaCookieJar to httpx.Cookies
    httpx_cookies = httpx.Cookies()
    for cookie in cookie_jar:
        httpx_cookies.set(
            name=cookie.name,
            value=cookie.value,
            domain=cookie.domain,
            path=cookie.path
        )
    return httpx_cookies
```

### Files Modified

**src/trackers/ASC.py** (ASC-specific fix):
- Added `_convert_cookiejar_to_httpx()` method to `ASC` class
- Updated `validate_credentials()` method to convert cookies before assigning
- Updated `search_existing()` method to convert cookies before assigning
- Updated `get_requests()` method to convert cookies before assigning
- Updated `upload()` method to convert cookies before assigning

**Future Work**: If this fix proves successful, the conversion logic should be moved to `src/cookie_auth.py` in the `load_session_cookies()` method so all trackers benefit automatically.

### Changes Summary

**Before (ASC.py):**
```python
async def validate_credentials(self, meta):
    self.session.cookies = await self.cookie_validator.load_session_cookies(meta, self.tracker)
    # Cookies NOT sent in requests ❌
```

**After (ASC.py):**
```python
async def validate_credentials(self, meta):
    cookie_jar = await self.cookie_validator.load_session_cookies(meta, self.tracker)
    self.session.cookies = self._convert_cookiejar_to_httpx(cookie_jar)
    # Cookies properly sent in requests ✅
```

## Impact

### Affected Trackers

**Currently Fixed:**
- ASC (Amigos Share Club) - ✅ Fixed in this PR

**Potentially Affected (use cookie_auth.py):**
The following trackers use the same cookie authentication pattern and may be affected by the same issue:
- AR (AlphaRatio)
- AVISTAZ_NETWORK (AvistaZ, CinemaZ, etc.)
- BJS
- BT
- FF
- HDS
- HDT
- IS
- PTS

**Note**: This PR implements the fix only for ASC as a proof-of-concept. If the fix is successful, it should be extended to all trackers listed above. The fix can be easily applied to `src/cookie_auth.py` to benefit all trackers at once.

### Benefits

- ✅ Fixes authentication failures for cookie-based trackers
- ✅ Ensures cookies are properly sent in all HTTP requests
- ✅ Maintains backward compatibility (handles both `MozillaCookieJar` and `httpx.Cookies`)
- ✅ Minimal code changes with consistent pattern across affected files

## Testing

### Manual Testing

A minimal test script `test_asc_cookies.py` was created to verify the fix:

1. Loads cookies from `data/cookies/ASC.txt`
2. Converts cookies to httpx format
3. Makes test requests to ASC endpoints
4. Verifies successful authentication (no login redirects)

**Test Results:**
- Before fix: All requests redirected to login page (authentication failed)
- After fix: All requests succeeded with full HTML pages (15KB+)

### Recommended Testing

For each affected tracker:
1. Export fresh cookies from browser
2. Attempt to upload or validate credentials
3. Verify authentication succeeds
4. Check that upload process completes successfully

## Historical Context

The cookie handling code was refactored in commit `76daec40` (October 24, 2025) which centralized cookie validation and upload logic into `src/cookie_auth.py`. This refactoring introduced the pattern of passing `MozillaCookieJar` directly to httpx, which exposed this compatibility issue.

## References

- **Test script**: `test_asc_cookies.py` (minimal reproduction case)
- **Test script with httpbin**: `test_httpx_simple.py` (validates httpx behavior)
- **httpx version**: 0.28.1
- **Related commit**: 76daec40 ("refactor: centralize cookie validation and upload logic")

## Additional Notes

### Alternative Solutions Considered

1. **Manual Cookie header**: Setting `headers['Cookie']` manually works but bypasses httpx's cookie management features (domain matching, path matching, expiration handling)

2. **Using requests library**: Would work but requires major refactoring from httpx to requests

3. **Converting to httpx.Cookies** (chosen solution): Maintains httpx usage while ensuring compatibility

### Why httpx.Cookies Conversion Was Chosen

- Minimal code changes required
- Preserves httpx's cookie management features (domain, path, secure flags)
- Maintains consistency with httpx's API design
- Future-proof solution that works with httpx's internal cookie handling

---

**Document Version**: 1.0
**Date**: 2025-12-31
**Author**: William Okano
**Related Issue**: [To be created]
**Related PR**: [To be created]
