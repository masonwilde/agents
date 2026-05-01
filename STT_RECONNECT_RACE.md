# STT reconnect race during session close

## Summary

When a Deepgram WebSocket drops shortly before or during agent session shutdown, `_main_task` retries the connection after the HTTP session has been closed, producing an unhandled `AttributeError`.

## Error

```
STTError(
  type='stt_error',
  label='livekit.plugins.deepgram.stt_v2.STTv2',
  error=AttributeError("'NoneType' object has no attribute 'getaddrinfo'"),
  recoverable=False
)
```

## Timeline

1. Deepgram WebSocket drops (transient network blip, Deepgram-side close)
2. `recv_task` checks `closing_ws` (False) and `self._session.closed` (False) -- both guards miss
3. `recv_task` raises `APIStatusError("deepgram connection closed unexpectedly")`
4. `_main_task` (`livekit-agents/livekit/agents/stt/stt.py:318`) catches `APIError`, sleeps, retries
5. Call ends naturally during the retry window -- shutdown begins
6. `job_proc_lazy_main.py:364` calls `_close_http_ctx()`, closing the shared `aiohttp.ClientSession` and setting `TCPConnector._resolver = None`
7. `_main_task` wakes, calls `_run()` -> `_connect_ws()` -> `session.ws_connect()` -> `getaddrinfo` -> `AttributeError`
8. `_main_task:344` catches `Exception`, emits `STTError(recoverable=False)`

## Why the guards don't help

- `closing_ws` is only set in `send_task` after the input channel closes and the `async for` loop exits naturally. If the WebSocket drops independently, `send_task` hasn't exited yet.
- `self._session.closed` is only True after `_close_http_ctx()` runs at shutdown step 8 of 8 (`job_proc_lazy_main.py:364`), long after the WebSocket drop.

## Why it's rare

Requires two independent events to overlap: a transient Deepgram failure AND a call ending during the retry window (a few seconds). Normal shutdown uses `CancelledError` (a `BaseException`), which bypasses `_main_task`'s `except APIError` retry logic entirely. The bug only fires when the retry loop is already active from an independent failure.

## Locations

| What | File | Line |
|------|------|------|
| `_connect_ws` narrow except | `livekit-plugins/livekit-plugins-deepgram/livekit/plugins/deepgram/stt_v2.py` | ~499 |
| `recv_task` session.closed guard | same file | ~403 |
| `_main_task` retry loop | `livekit-agents/livekit/agents/stt/stt.py` | ~313 |
| `_close_http_ctx` call | `livekit-agents/livekit/agents/ipc/job_proc_lazy_main.py` | ~364 |
| `cancel_and_wait` (no timeout) | `livekit-agents/livekit/agents/utils/aio/utils.py` | ~6 |

## Fix options

**1. Guard in `_connect_ws` (minimal, prevents the `AttributeError`):**
```python
async def _connect_ws(self) -> aiohttp.ClientWebSocketResponse:
    if self._session.closed:
        raise APIConnectionError("http session is closed, cannot reconnect")
    ...
```

**2. Guard in `_main_task` retry (prevents retrying during shutdown):**
```python
except APIError as e:
    if self._input_ch.closed:
        raise  # shutting down, don't retry
    ...
```

**3. Broaden the except in `_connect_ws`:**
```python
except (aiohttp.ClientConnectorError, asyncio.TimeoutError, AttributeError) as e:
    raise APIConnectionError("failed to connect to deepgram") from e
```

Fix 1 is the cleanest. Fix 2 prevents unnecessary work. Fix 3 is a safety net.

## Related

- [#5497](https://github.com/livekit/agents/issues/5497) -- `_aclose_impl` hangs indefinitely when participant disconnects mid-speech (same root cause area: unbounded/uncoordinated shutdown)
