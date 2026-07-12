<img src="https://my-badges.github.io/my-badges/fix-4.png" alt="I did 4 sequential fixes." title="I did 4 sequential fixes." width="128">
<strong>I did 4 sequential fixes.</strong>
<br><br>

Commits:

- <a href="https://github.com/ZuBB/chat/commit/9686f9e8e936b740ccd49c9130b66c61da74d995">9686f9e</a>: fix: handle internal reconnect failures

Route ping timeouts and unexpected socket closures through a shared fire-and-forget helper. Reconnect rejections are now emitted through the client error event instead of reaching the runtime as unhandled promises.
- <a href="https://github.com/ZuBB/chat/commit/309169086852d4320d01843108e689ebb934fa63">3091690</a>: fix: cancel reconnects on explicit close

Separate internal socket replacement from public shutdown so close() can cancel a pending backoff without reconnect() cancelling itself. Install cancellation before emitting reconnecting to cover synchronous shutdown listeners.
- <a href="https://github.com/ZuBB/chat/commit/ee2e20b4a8d6ced37c4d6eb77e3a25cf78191d09">ee2e20b</a>: fix: coalesce concurrent reconnect requests

Keep one active reconnect promise and return it to overlapping callers. This prevents normal lifecycle races from creating duplicate timers or surfacing an already-reconnecting error.
- <a href="https://github.com/ZuBB/chat/commit/960b3d51658e588427132a781eb9be71c56289e8">960b3d5</a>: fix: ignore events from stale sockets

Bind WebSocket callbacks to the instance that registered them. Delayed events from a replaced socket can no longer mutate the active connection or start another reconnect cycle.

Add a fake-WebSocket regression test and a minimal Node test command.


Created by <a href="https://github.com/my-badges/my-badges">My Badges</a>