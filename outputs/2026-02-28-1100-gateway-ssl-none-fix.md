> **Question:** While configuring settings in Mission Control ssl=None is incompatible with a wss:// URI

## Root Cause

`websockets==16.0` (breaking change from v14+) made `ssl=None` explicitly invalid for `wss://` URIs.

In `gateway_rpc.py`, `_create_ssl_context` returns `None` for a standard `wss://` connection (when `allow_insecure_tls=False`). Both `_openclaw_call_once` (line 409) and `_openclaw_connect_metadata_once` (line 425) then call:

```python
websockets.connect(gateway_url, ssl=ssl_context, **connect_kwargs)
```

…where `ssl_context` is `None`. `websockets` v16 raises `ValueError: ssl=None is incompatible with a wss:// URI`.

## Fix

Move `ssl` into `connect_kwargs` conditionally — only pass it when a custom (insecure) context is needed. When `ssl_context is None`, omit the kwarg entirely and let `websockets` infer SSL from the URI scheme (`wss://` → SSL on, `ws://` → SSL off).

**Before (both call sites):**
```python
ssl_context = _create_ssl_context(config)
connect_kwargs: dict[str, Any] = {"ping_interval": None}
if origin is not None:
    connect_kwargs["origin"] = origin
async with websockets.connect(gateway_url, ssl=ssl_context, **connect_kwargs) as ws:
```

**After:**
```python
ssl_context = _create_ssl_context(config)
connect_kwargs: dict[str, Any] = {"ping_interval": None}
if origin is not None:
    connect_kwargs["origin"] = origin
if ssl_context is not None:
    connect_kwargs["ssl"] = ssl_context
async with websockets.connect(gateway_url, **connect_kwargs) as ws:
```

The existing test `test_create_ssl_context_returns_none_for_wss_with_secure_mode` remains valid — `_create_ssl_context` still returns `None` for that case. The change is only in how the `None` return value is handled at the two call sites.
