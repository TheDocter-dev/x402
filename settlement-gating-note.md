# Settlement-gating note: paid 3xx responses delivered without settlement (x402 Python, v1 and v2 Flask < 2.15.0)

**Status:** DRAFT — prepared for the x402 maintainers; not reviewed or endorsed by them. Lives at https://github.com/TheDocter-dev/x402/blob/settle-gating-note/settlement-gating-note.md until they choose where it should live.
**Tracking issue:** x402-foundation/x402#3465
**Date:** 2026-09-14

## Summary

In the x402 Python SDK, the affected server middlewares settle payment only
when the protected route returns a 2xx response. If a paid route returns a 3xx
response — most commonly a 302 redirect or a 304 Not Modified — the response
is delivered to the client **and the payment is never settled**. The client has
been verified as having paid, receives the resource, and no funds move.

Affected: the deprecated v1 line (published on PyPI as `x402==1.0.0`,
2025-12-10) in **both** the Flask and FastAPI adapters, and the v2 **Flask**
middleware for all releases `>= 2.0.0, < 2.15.0`. The v2 FastAPI middleware
settled `< 400` from 2.0.0 onward and is not affected. The v2 Flask gate was
fixed in **2.15.0** via PR #2826, which settles for any response status
`< 400`.

## Who is affected

You are exposed if **both** of these are true:

1. Your service depends on `x402==1.0.0` (PyPI, either adapter), or uses the
   v2 **Flask** middleware on `x402 >= 2.0.0, < 2.15.0`; and
2. Any handler behind the payment middleware can return a 3xx response —
   redirects (`301/302/303/307/308`), `304 Not Modified`, or any framework
   behavior that produces them (trailing-slash redirects, conditional GET with
   ETag/Last-Modified).

Quick self-check: search your codebase for handlers on paid paths that call
`redirect(`, return `RedirectResponse`, set a 3xx status, or rely on
framework-level redirects. Conditional-GET support (ETags) on a paid path is
enough to trigger the 304 case.

## Technical detail

### v1 Flask (`x402==1.0.0` wheel)

`x402/flask/middleware.py:294-296` (wheel; `:294-296` on current main):

```python
if (
    response_wrapper.status_code is not None
    and response_wrapper.status_code >= 200
    and response_wrapper.status_code < 300
):
    # Settle the payment for successful responses
```

Unlike v2, v1 Flask buffers the full response body before the settle decision,
so on the non-settle path the client receives the complete paid payload.

### v1 FastAPI (`x402==1.0.0` wheel)

`x402/fastapi/middleware.py:197` (wheel; `:201` on current main):

```python
# Early return without settling if the response is not a 2xx
if response.status_code < 200 or response.status_code >= 300:
    return response
```

### v2 Flask (< 2.15.0)

The v2 Flask middleware gated settlement on `200 <= status < 300`
(`x402/http/middleware/flask.py:414` in the 2.0.0 wheel; `:438-439` in the
2.10.0 wheel). Fixed in **2.15.0** (PR #2826): settlement proceeds for any
status `< 400` (`flask.py:420`, cancel path `:408`; current main
`flask.py:486`), with a regression test at
`python/x402/tests/unit/http/middleware/test_flask.py:788`.

The v2 FastAPI middleware settles unless the status is `>= 400`
(`x402/http/middleware/fastapi.py:317` in the 2.0.0 wheel; `:337` in 2.10.0)
and was not affected at any v2 release. v2 ships exactly these two middleware
adapters, so this boundary map is exhaustive.

### Verification

Executed reproduction against the published wheels (mocked facilitator
counting `verify`/`settle` calls, real WSGI/ASGI stacks, zero on-chain spend),
by two parties, independently:

| adapter (wheel)        | paid route returns | verify | settle | client receives |
|------------------------|--------------------|--------|--------|-----------------|
| Flask 1.0.0            | 302 + Location     | 1      | 0      | 302 + redirect body |
| FastAPI 1.0.0          | 302 + Location     | 1      | 0      | 302             |
| Flask 1.0.0 (control)  | 200                | 1      | 1      | 200 + X-PAYMENT-RESPONSE |
| Flask 1.0.0 (304 case) | 304                | 1      | 0      | 304             |

- Settle (@TheDocter-dev), 2026-09-14 — wheel `x402-1.0.0-py3-none-any.whl`,
  sha256 `9ae2d2fac5045ff8b18f70ee2033c52659ebdbaff4d07dae3f3f07c96611218b`;
  version-boundary wheels 2.0.0 (sha256
  `9db08530b9daa0a00d8505f7cd0bddf45a27ff8948d11bb9961d324d10b91a50`),
  2.10.0 (sha256 `7e387e6d8c7fa706ffd35754da28e75b8493775d81bc6888cad0d9439106295d`),
  2.15.0 (sha256 `06f59422d5ca30736964ce911980fe3288ed72ab90b14789a42738fd5e99b794`).
- @halobartku, 2026-09-14 — independent harness, offered in #3465.

Note: the v1 adapters read the payment header as **`X-PAYMENT`** (both Flask
and FastAPI, `request.headers.get("X-PAYMENT")`), not the v2 `X-402-PAYMENT` —
grep for the right string when auditing v1 services.

## Mitigation

- **Upgrade to `x402 >= 2.15.0`.** This is the only complete fix; v1 is
  deprecated and this behavior is part of what the v2 rewrite corrected.
- If you cannot upgrade immediately: eliminate 3xx from paid paths (no
  redirects behind the middleware, disable conditional GET on paid paths), or
  wrap the middleware and invoke settlement yourself for any response status
  `< 400`.

## Scope note: 304 semantics

Whether a 304 *should* settle is a cross-adapter policy question (a 304 carries
no body, but the client still got a usable response for a paid resource). That
decision belongs to the maintainers and is tracked in #3465; this note only
documents the behavior that ships today.

## Credits

- Initial report and version-boundary map: Settle (@TheDocter-dev)
- Independent wheel-level reproduction, FastAPI v1 path discovery, response-body
  buffering observation, and `X-PAYMENT` header correction: @halobartku
- v2 fix: @phdargen (PR #2826)
