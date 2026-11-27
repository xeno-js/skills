# Shared utilities

Technical reference assembled from the website documentation. Each section
identifies its original Markdown source. Website frontmatter and the repeated
sponsorship footer are omitted; body content and fenced examples are retained.

---

## Contents

- [`docs/utils/date.md`](#source-1)
- [`docs/utils/enumerable.md`](#source-2)
- [`docs/utils/guards.md`](#source-3)
- [`docs/utils/guid.md`](#source-4)
- [`docs/utils/http.md`](#source-5)
- [`docs/utils/math.md`](#source-6)
- [`docs/utils/overview.md`](#source-7)
- [`docs/utils/promise.md`](#source-8)
- [`docs/utils/route.md`](#source-9)
- [`docs/utils/sanitize.md`](#source-10)
- [`docs/utils/string.md`](#source-11)

<a id="source-1"></a>

# Source: `docs/utils/date.md`

## Format and compare dates

`DateHelper` works with JavaScript `Date` objects and timestamps. Import it from
`@xeno-js/core` or `@xeno-js/shared`:

```ts
import { DateHelper } from '@xeno-js/core'

const createdAt = new Date('2026-01-01T12:00:00.000Z')
const expiresAt = DateHelper.addDays(createdAt, 2)

console.log(DateHelper.toISOString(expiresAt)) // 2026-01-03T12:00:00.000Z
console.log(DateHelper.isAfter(expiresAt, createdAt)) // true
console.log(DateHelper.isExpired(expiresAt, expiresAt.getTime())) // false
```

## Methods

- `toISOString(date)` delegates to `Date.toISOString()` and returns a UTC
  string. An invalid date throws.
- `addDays(date, days)` returns a new date by adding `days * 86400000`
  milliseconds. Negative and fractional values are accepted; the original date
  is not modified.
- `isExpired(expiresAt, nowMs?)` compares the expiry timestamp with `nowMs`, or
  `Date.now()` when omitted. Equality is not expired.
- `isAfter(after, before)` returns `true` only for valid dates where the first
  is strictly later.
- `isFuture(date)` returns `true` only for a valid date strictly later than the
  current time.

## Common problems

Adding 24 hours is not the same as adding a local calendar day across a
daylight-saving transition. Use calendar-aware logic when local wall-clock time
must stay constant. `isExpired()` does not validate its date; an invalid date
produces a `false` comparison, so validate untrusted dates with
[Guards](/docs/utils/guards) first.

---

---

<a id="source-2"></a>

# Source: `docs/utils/enumerable.md`

## Find the first array item

`Enumerable` provides two ways to retrieve the first item of a mutable array,
optionally using a predicate:

```ts
import { Enumerable } from '@xeno-js/core'

const products = [
  { name: 'Keyboard', available: false },
  { name: 'Mouse', available: true },
]

const firstAvailable = Enumerable.first(
  products,
  (product) => product.available,
)
const missing = Enumerable.firstOrDefault(
  products,
  (product) => product.name === 'Monitor',
)

console.log(firstAvailable.name) // Mouse
console.log(missing) // undefined
```

`first(array, predicate?)` throws `Error('Sequence contains no elements.')` if
the selected item is not defined. `firstOrDefault(array, predicate?)` returns
`undefined` instead. Without a predicate, each method checks the first element
only; neither searches past an undefined first element.

## Common problems

Both methods treat `null`, `undefined`, and `NaN` as missing because they use
`Guards.isDefined()`. A predicate that matches such a value does not make it a
valid result. Use `firstOrDefault()` when absence is expected, and handle its
`undefined` return value.

---

---

<a id="source-3"></a>

# Source: `docs/utils/guards.md`

## Check values at runtime

`Guards` provides predicates and assertions for common JavaScript values. Type
predicates such as `isString()` can also narrow an `unknown` value in
TypeScript.

```ts
import { Guards } from '@xeno-js/core'

function displayName(value: unknown): string {
  if (!Guards.isString(value) || Guards.isNullOrEmpty(value)) {
    throw new Error('Name is required')
  }
  return value.trim()
}

console.log(displayName('  Ada  ')) // Ada
console.log(Guards.isNullOrEmpty(false)) // false
console.log(Guards.isDefined(NaN)) // false
```

## Available checks

- `isDefined(value)` rejects `null`, `undefined`, and `NaN`.
- `isNullOrEmpty(value)` also treats whitespace-only strings and empty arrays as
  empty. It does not treat `false`, `0`, or an empty object as empty.
- `isString`, `isNumber`, `isInteger`, `isBoolean`, `isBigInt`, `isSymbol`, and
  `isFunction` check primitive or function types. `isNumber` requires a finite
  number.
- `isArray<T>()` checks that the value is an array, not that its elements are of
  type `T`.
- `isDate()` checks for a valid `Date`; `isError()` uses `instanceof Error`.
- `isObject()` accepts non-null objects, including arrays. `isObjectRecord()`
  checks the `[object Object]` tag; it is not a prototype or schema validation.
- `isPromiseLike()` checks for a callable `then`; `hasMethod(object, name)`
  checks for a callable property.

## Assertions

`throwIfNullOrEmpty(value, message)` throws the supplied message for an empty
value. `throwIfNotInteger(value, message)` throws unless the value is a finite
integer.

`throwIfNegative(value, message)` throws only for negative integers. Negative
fractions and non-integers pass this particular check; validate integer
requirements separately when needed.

```ts
Guards.throwIfNotInteger(3, 'Count must be an integer')
Guards.throwIfNegative(3, 'Count must not be negative')
```

## Common problems

Type guards do not validate nested fields or business rules. In particular, a
generic argument to `isArray<T>()` is not a runtime element validator. Validate
each item or use an application schema for untrusted input.

---

---

<a id="source-4"></a>

# Source: `docs/utils/guid.md`

## Generate and validate identifiers

`GuidHelper.generate()` calls `crypto.randomUUID()` to generate a UUID v4. The
runtime must provide the global Web Crypto API.

```ts
import { GuidHelper } from '@xeno-js/core'

const id = GuidHelper.generate()
console.log(GuidHelper.isValid(id)) // true

const parsed = GuidHelper.parse('550e8400-e29b-41d4-a716-446655440000')
if (parsed === undefined) throw new Error('Invalid identifier')
```

## Methods

- `generate()` returns a random UUID v4.
- `isValid(value)` checks the UUID v4 string format, case-insensitively.
- `parse(value)` returns the original valid string or `undefined`; it does not
  normalize case.
- `isValidGuid(value)` checks a `Guid` value by converting it to a string and
  checking validity and emptiness.
- `isEmpty(value)` checks for the exact all-zero GUID string. Other invalid
  strings are not considered empty.

## Common problems

A valid UUID of another version does not pass `isValid()`. Validate external
strings before treating them as identifiers. The UUID format check does not
establish that an identifier exists or that the caller may access the associated
resource.

---

---

<a id="source-5"></a>

# Source: `docs/utils/http.md`

## Work with HTTP data

`HttpHelper` provides transformations used by Xeno.JS HTTP components. For the
response contract and related examples, see
[HTTP helpers](/docs/http/helper-http) and
[API response DTOs](/docs/http/api-response).

```ts
import { HttpHelper } from '@xeno-js/core'

const headers = HttpHelper.normalizeHeaders({
  'X-Request-Id': 42,
  'Accept': ['application/json', 'text/plain'],
})
// { 'X-Request-Id': '42', Accept: 'application/json,text/plain' }

const cookies = HttpHelper.parseCookies('theme=dark; name=Ada%20Lovelace')
// { theme: 'dark', name: 'Ada Lovelace' }

const response = HttpHelper.success({ id: 42 }, 201)
```

## Headers, cookies, and URLs

- `normalizeHeaders(headers)` converts defined object values to strings and
  joins arrays with commas. It preserves key casing and omits `null`,
  `undefined`, and `NaN`; a non-object input produces `{}`.
- `parseCookies(header)` splits a cookie header on semicolons and decodes each
  value. The last duplicate name wins; malformed percent-encoding is kept as-is.
- `sanitizeOriginUrl(url)` returns the `URL.origin` of a parseable absolute URL,
  or `undefined` if parsing fails. Some opaque origins produce the string
  `"null"`.
- `parseSafeUrl(url, fallbackHost?)` tries an absolute URL, then resolves
  against the fallback, which defaults to `http://localhost`. Invalid input or
  an invalid fallback can still throw.
- `maskIp(ip)` replaces the last dot-separated segment of a four-part address or
  the last colon-separated segment. Other inputs can be returned unchanged.

## Bodies and response DTOs

- `parseBody(method, rawBody)` handles `POST`, `PUT`, `PATCH`, and `DELETE`,
  case-insensitively. It passes through supported body types, serializes
  objects, and converts other defined values to strings. Other methods or
  missing inputs return `undefined`.
- `success(data, status?, meta?, customHeaders?)` returns a `ResponseDto` with
  `ok: true` and a `{ success: true, data, meta }` payload. The status defaults
  to `200`. The JSON `Content-Type` overrides the same-cased custom header.
- `error(dto, status?, customHeaders?)` builds an error DTO with `ok: false`,
  identifiers, a new timestamp, and no-cache headers. The status defaults to
  `500`; custom headers can override default headers.
- `resolveFormatIndicator(accept, contentType)` checks the first Accept
  candidate, then Content-Type, then falls back to `application/json`. It does
  not negotiate quality values or inspect later Accept candidates.
- `normalizeMediaType(value, fromAccept)` normalizes the first candidate and
  removes parameters. `isValidMediaType(mediaType)` checks the media-type syntax
  used by the helper.

## Security limitations

URL parsing does not apply an origin or host allowlist and does not prevent SSRF
or open redirects. Validate the scheme and destination for your use case.
`maskIp()` is a simple string transformation, not reliable anonymization. Header
conversion does not validate header names or values for your transport.

---

---

<a id="source-6"></a>

# Source: `docs/utils/math.md`

## Numeric operations

```ts
import { MathHelper } from '@xeno-js/core'

const quantity = MathHelper.clamp(15, 0, 10) // 10
const ratio = MathHelper.safeDivide(5, 0, -1) // -1
const completion = MathHelper.toPercentage(3, 4) // 75
const amount = MathHelper.toNumber('12.5') // 12.5
```

## Methods

- `clamp(value, min, max)` applies inclusive bounds with `Math.min` and
  `Math.max`. Supply `min <= max`; the helper does not validate the bounds.
- `roundTo(value, decimals)` scales by a power of ten and uses `Math.round`.
- `safeDivide(numerator, denominator, fallback?)` returns the fallback, default
  `0`, only when the denominator is zero.
- `toPercentage(part, total)` returns `part / total * 100`, or `0` when the
  total is zero.
- `toNumber(value, fallback?)` uses `Number()` for conversion and returns the
  fallback, default `0`, for `null`, `undefined`, or `NaN` results.

## Common problems

JavaScript floating-point rounding is not exact decimal arithmetic. For monetary
calculations, choose a representation and rounding policy appropriate for your
application.

`toNumber()` can return infinity, and `Number()` can throw for some inputs,
including symbols. Empty strings convert to zero. Validate inputs and require a
finite result when those behaviors are unsuitable.

---

---

<a id="source-7"></a>

# Source: `docs/utils/overview.md`

## Shared utilities in Xeno.JS

Xeno.JS exports small utilities for common operations from `@xeno-js/shared`.
The same exports are available through `@xeno-js/core`, so an application
already using core can import them there:

```ts
import { DateHelper, Guards, GuidHelper } from '@xeno-js/core'

const id = GuidHelper.generate()
const expiresAt = DateHelper.addDays(new Date(), 1)

if (Guards.isDefined(id)) {
  console.log(id, expiresAt)
}
```

Each helper provides a specific operation. Check its inputs and edge cases
before using it as part of validation, security, or persistence logic.

## Utility reference

- [DateHelper](/docs/utils/date): UTC formatting, date arithmetic, and
  comparisons.
- [Enumerable](/docs/utils/enumerable): find the first array item.
- [Guards](/docs/utils/guards): runtime predicates and assertions.
- [GuidHelper](/docs/utils/guid): generate and validate UUID v4 identifiers.
- [HttpHelper](/docs/utils/http): headers, cookies, URLs, request bodies, and
  response DTOs.
- [MathHelper](/docs/utils/math): numeric conversions and arithmetic.
- [PromiseHelper](/docs/utils/promise): asynchronous delays and jitter.
- [Route helpers](/docs/utils/route): route patterns and request parameter
  extraction.
- [SanitizeHelper](/docs/utils/sanitize): control-character removal and limited
  sanitization.
- [StringHelper](/docs/utils/string): JSON conversion, formatting, and header
  values.

---

---

<a id="source-8"></a>

# Source: `docs/utils/promise.md`

## Wait asynchronously

`delay(ms)` resolves after a timer. `delayWithJitter(baseDelayMs, maxJitterMs)`
adds `Math.floor(Math.random() * maxJitterMs)` milliseconds to the base delay.

```ts
import { PromiseHelper } from '@xeno-js/core'

await PromiseHelper.delay(100)
await PromiseHelper.delayWithJitter(100, 50)
```

For positive integer `maxJitterMs`, the second call schedules a delay from 100
through 149 milliseconds. Jitter can spread retry attempts across clients, but
this helper does not perform retries. For client resilience configuration, see
[HTTP Client](/docs/http/client).

## Common problems

Both helpers use `setTimeout`: event-loop scheduling can delay completion, and
there is no cancellation or exact timing guarantee. Validate delay values before
use. `Math.random()` is not suitable for cryptographic randomness.

---

---

<a id="source-9"></a>

# Source: `docs/utils/route.md`

## Parse route patterns

`RoutePatternHelper` normalizes paths and constructs regular expressions.
Dynamic segments use `:name` or `{name}`:

```ts
import { RoutePatternHelper, HttpRouteParamHelper } from '@xeno-js/core'

const route = RoutePatternHelper.parse('/users/:id')
console.log(route.isDynamic) // true
console.log(route.pattern.test('/users/42')) // true

const params = HttpRouteParamHelper.extractPathParams('/users/:id', '/users/42')
// { id: '42' }
```

- `normalizePath(path)` trims whitespace, adds a leading slash if needed, and
  removes trailing slashes except for the root path. Repeated internal slashes
  are not collapsed.
- `isDynamicPath(path)` checks for a colon or both braces anywhere in the path.
- `createRoutePattern(path)` creates a regular expression from path segments,
  escaping static segments and capturing dynamic ones.
- `parse(path)` returns `{ pattern, isDynamic }`. Dynamic paths use
  `createRoutePattern`; static paths are inserted into a regular expression
  without escaping metacharacters.

Use trusted route definitions. For static paths containing regex metacharacters,
account for their regex meaning or construct a suitably escaped pattern
yourself.

## Extract request data

- `extractPathParams(routePattern, requestPath)` extracts dynamic segments by
  position, decodes them, and strips control characters. It does not verify
  static segments or route matching. Invalid percent-encoding can throw.
- `extractQueryParams(requestUrl)` parses the URL, strips control characters,
  and limits keys to 64 and values to 512 characters. Later duplicate keys
  overwrite earlier ones.
- `extractInput<TInput>(req, routePattern)` merges the parsed body, `req.query`,
  and path parameters in that order. Path values take precedence. `TInput` is a
  compile-time type, not a runtime schema.

For `POST`, `PUT`, and `PATCH`, `extractInput()` reads JSON, URL-encoded form
data, or text according to Content-Type. It catches body-reading and parsing
failures and uses `undefined` as the body. Structured body values are
recursively sanitized, including array elements; `req.query` is used directly.
Validate the merged data before passing it to application logic.

See [HTTP routes](/docs/http/routes) for route registration.

---

---

<a id="source-10"></a>

# Source: `docs/utils/sanitize.md`

## Remove control characters

`SanitizeHelper` provides limited string and object transformations. Validate
the input type and apply rules for the destination where the data will be used.

```ts
import { SanitizeHelper } from '@xeno-js/core'

const name = SanitizeHelper.stripControlChars('  Ada\nLovelace  ')
// AdaLovelace

const path = SanitizeHelper.sanitizePath('javascript:alert(1)')
// /

const roles = SanitizeHelper.sanitizeStringArray([' admin\n', ' ', 'reader'])
// ['admin', 'reader']
```

## Methods

- `stripControlChars(value, maxLength?)` removes ASCII control characters, trims
  whitespace, and truncates to 256 characters by default. It returns `undefined`
  for an absent or empty result.
- `sanitizePath(path, maxLength?)` removes controls and trims, then limits the
  result to 512 characters by default. Initially empty input returns
  `undefined`; control-only input and `javascript:`, `data:`, or `vbscript:`
  prefixes return `/`.
- `sanitizeStringArray(items, maxItemLength?, maxItems?)` considers the first 50
  items by default, strips controls from each with a default length of 64, and
  removes empty results. An absent or initially empty array returns `undefined`;
  a non-empty array can produce `[]`.
- `sanitizeObject(obj)` walks object properties recursively, sanitizes string
  values, and omits `__proto__`, `constructor`, and `prototype` keys. Nested
  arrays are kept without sanitizing their contents. Non-object input returns
  `{}`.

## Security limitations

These helpers are not HTML escaping, SQL parameterization, schema validation, or
a redirect allowlist. `sanitizePath()` can still return absolute URLs and other
untrusted destinations. `sanitizeObject()` does not handle cyclic objects and
does not sanitize array elements. Apply context-specific validation and encoding
at the boundary.

---

---

<a id="source-11"></a>

# Source: `docs/utils/string.md`

## Format and parse strings

```ts
import { StringHelper } from '@xeno-js/core'

console.log(StringHelper.camelCase('order-status')) // orderStatus
console.log(StringHelper.interpolate('Hello {{name}}', { name: 'Ada' })) // Hello Ada
console.log(StringHelper.safeParse('not-json', { enabled: false })) // { enabled: false }
console.log(StringHelper.getSingleValue(['application/json', 'text/plain'])) // application/json
```

## JSON helpers

- `safeStringify(value)` uses `JSON.stringify()` and falls back to
  `String(value)` if serialization throws. `String()` itself can also throw for
  some values. Despite the declared string return type,
  `JSON.stringify(undefined)` returns `undefined` at runtime.
- `safeParse<T>(input, fallback?)` returns the parsed value on success. On
  invalid JSON it returns the non-nullish fallback, or the original input when
  no fallback is supplied. Validate the parsed shape before using it as `T`.
- `safeStringifyObject(obj)` applies `safeStringify()` to each defined own
  enumerable property, skipping `null`, `undefined`, and `NaN`.

## String and header helpers

- `camelCase(input)` splits on hyphens, underscores, and whitespace, lowercases
  the first segment, and capitalizes the first character of subsequent segments.
  It does not split existing camelCase words.
- `interpolate(template, vars)` replaces `{{key}}` tokens for defined values.
  Unknown tokens remain unchanged.
- `truncate(input, maxLength, suffix?)` returns the original string if short
  enough; otherwise it appends the suffix, which defaults to `…`. If the suffix
  is longer than `maxLength`, the result can exceed that limit.
- `generateReferenceCode(prefix)` produces a prefix, six random alphanumeric
  characters, and the local year. It uses `Math.random()` and does not guarantee
  uniqueness.
- `getSingleValue(value, separator?)` returns a string or the first array item.
  With a separator, it returns the first part; an empty array returns
  `undefined`.
- `getSingleValueWithSplit(value, separator)` splits and trims the first part
  only when the separator occurs in the value.

## Common problems

Neither interpolation nor JSON parsing escapes content for HTML, SQL, or other
destinations. Apply validation and context-specific encoding. Use a
cryptographic identifier generator, such as [GuidHelper](/docs/utils/guid), when
a random UUID is required.

---
