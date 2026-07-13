<img src="https://my-badges.github.io/my-badges/fix-6+.png" alt="I did 9 sequential fixes." title="I did 9 sequential fixes." width="128">
<strong>I did 9 sequential fixes.</strong>
<br><br>

Commits:

- <a href="https://github.com/ZuBB/chat/commit/3956f3d23a248b526a1b761462426a94b52038da">3956f3d</a>: fix: contain IRC protocol processing failures

Problem:
IRC parsing and message handlers ran directly inside the WebSocket message callback, and unsolicited PONG handling threw when no client ping had been recorded.

Impact:
One malformed or unexpected line could escape the callback as an unhandled exception and prevent valid later lines in the same frame from being processed.

Cause:
The transport had no per-line protocol error boundary or structured protocol error type.

Change:
Export ProtocolError with bounded sanitized line context, catch parsing and handler failures independently for every IRC line, emit them through the existing error event, and continue processing subsequent lines without forcing reconnect for recoverable input.

Compatibility:
Valid IRC handling and existing events are unchanged; protocol failures become observable Error subtypes instead of callback exceptions.

Tests:
Send an unsolicited PONG followed by valid authentication in one frame and verify the protocol error is reported while readiness still completes.
- <a href="https://github.com/ZuBB/chat/commit/5eda089b5d346bcbfe46a6a6e4fb5d51dd99a282">5eda089</a>: fix: adapt IRC command acknowledgement deadlines

Problem:
Command responders defaulted to a one-second deadline whenever no client-originated ping latency had been recorded.

Impact:
Normal Twitch processing or temporary runtime latency could be misclassified as failed delivery, encouraging retries with uncertain delivery status and masking real connection-health signals.

Cause:
The timeout used twice a single optional RTT sample with a one-second floor; before keepalive produced a sample, the hardcoded fallback always selected that floor.

Change:
Export CommandTimeoutError with command, channel, generation, and duration context, use a five-second pre-RTT default, maintain a smoothed RTT, and calculate RTT-plus-margin deadlines bounded to three through fifteen seconds with additive tuning options.

Compatibility:
Explicit internal timeout overrides and existing error messages remain compatible; the new Error subtype and options are additive.

Tests:
Verify configured acknowledgement timing and typed timeout metadata.
- <a href="https://github.com/ZuBB/chat/commit/949272f8e41eddb6dd516a7a4945ce2de32fd13a">949272f</a>: fix: preserve and jitter reconnect backoff

Problem:
Reconnect attempts reset when the raw WebSocket opened, even if IRC authentication or channel restoration immediately failed, and every client used the same deterministic delay.

Impact:
Repeated handshake failures retried at the minimum interval and multiple clients could reconnect in lockstep during shared outages.

Cause:
Backoff success was defined at the socket layer rather than stable application readiness, and the exponential delay had no jitter.

Change:
Reset attempts only after 30 seconds of stable ready state, add configurable equal jitter with injectable randomness, preserve the existing base, multiplier, and cap defaults, and report the actual scheduled delay.

Compatibility:
Existing reconnect events and keepalive attempt counters remain available; all tuning options are additive.

Tests:
Verify equal-jitter bounds and cancellation plus delayed attempt reset after stable readiness.
- <a href="https://github.com/ZuBB/chat/commit/401b85f8d713a3ce0d20bac580215487496e04d1">401b85f</a>: fix: await authenticated reconnect completion

Problem:
The reconnect promise resolved as soon as a replacement WebSocket attempt was created, before Twitch IRC authentication or channel restoration succeeded.

Impact:
Callers could treat a still-authenticating or failed replacement as recovered and resume work against an unusable connection.

Cause:
The reconnect delay callback resolved immediately after invoking connect and exposed no readiness waiter.

Change:
Keep reconnect source-compatible while resolving it only after the connected authentication state, retain single-flight behavior and cancellation, and add waitUntilReady for consumers that require restored channel membership.

Compatibility:
The method signature is unchanged; its promise now provides the stronger completion guarantee implied by its name. waitUntilReady is additive.

Tests:
Verify reconnect remains unsettled before numeric 376, resolves after authentication, supports readiness waiting, and preserves concurrent and stale-socket behavior.
- <a href="https://github.com/ZuBB/chat/commit/75f5eb1c5c95d39c1f44567c6580e1c0079bfafd">75f5eb1</a>: fix: report and classify authentication failures

Problem:
Twitch login rejection silently closed the client as if the caller requested shutdown, malformed authentication used an imprecise generic error, and token-provider failures lacked authentication context.

Impact:
Consumers could not refresh or replace credentials reliably, terminal invalid credentials were invisible, and server-driven failures suppressed meaningful recovery decisions.

Cause:
Untagged authentication NOTICE handling called the public close path before classifying the server response, while asynchronous token failures propagated as generic errors.

Change:
Export AuthenticationError with transient classification, report secret-safe login, formatting, provider, and token-timeout failures, recover transient setup failures, and suppress automatic retry for terminal invalid credentials without marking them as intentional user closure.

Compatibility:
The existing error event remains the delivery mechanism; the exported Error subtype and classification are additive.

Tests:
Cover terminal Twitch login rejection without reconnect and transient token-provider failure with recovery.
- <a href="https://github.com/ZuBB/chat/commit/a7a7e79cc9a184ae43d656d2cff24fda9d1894e2">a7a7e79</a>: fix: isolate asynchronous socket generations

Problem:
A socket open handler could await an asynchronous token while a reconnect replaced that socket, then continue authentication using the client's new socket reference.

Impact:
Late work from a stale connection could send credentials through or mutate the lifecycle of a replacement connection, creating duplicate or corrupted handshakes.

Cause:
Socket identity was checked only before entering the asynchronous open handler and not after awaited token resolution.

Change:
Carry the opened socket through connection setup, revalidate it after asynchronous boundaries and before authentication writes, and route current-generation setup failures through observable recovery while ignoring stale failures.

Compatibility:
Public APIs are unchanged; only stale asynchronous work is suppressed.

Tests:
Delay token resolution, replace the socket, and verify the stale handler cannot authenticate the replacement.
- <a href="https://github.com/ZuBB/chat/commit/a19f46dfa75e23cfe90244c8fcbd8c664823d727">a19f46d</a>: fix: cancel command waits when connections close

Problem:
Pending join, part, and message acknowledgement waits survived socket closure until their independent command deadlines expired.

Impact:
Callers received delayed USERSTATE or command timeout errors that hid the actual network failure and held listeners after the responsible connection was gone.

Cause:
Command responders were not associated with a connection generation and socket closure did not notify them.

Change:
Export ConnectionClosedError, bind each wait to its generation, cancel matching waits immediately on close, preserve close code and reason, and remove listeners and timers exactly once.

Compatibility:
Successful command behavior is unchanged; failed operations now reject earlier with a more precise exported Error subtype.

Tests:
Verify a pending send rejects immediately with its abnormal close code and network reason.
- <a href="https://github.com/ZuBB/chat/commit/c3ef8109ec63db2e5875a15b8e975b5f24927842">c3ef810</a>: fix: bound connection lifecycle phases

Problem:
WebSocket opening, token resolution, IRC authentication, and channel joining had no explicit lifecycle model and could remain pending indefinitely.

Impact:
A connection attempt could stall without entering reconnect recovery, leaving callers unable to distinguish transport-open, authenticated, joined, and ready states.

Cause:
The client only exposed socket readiness and event callbacks; phase deadlines and state transitions were not represented.

Change:
Add compatible phase timeout options, enforce deadlines for socket open, token resolution, authentication, and join, and expose additive lifecycle states through stateChange events with connection generations.

Compatibility:
Existing connect and event APIs remain available. New options and the state event are additive.

Tests:
Cover authentication lifecycle transitions, bounded authentication recovery, and timeout cleanup alongside the existing reconnect suite.
- <a href="https://github.com/ZuBB/chat/commit/2807325eeb71c7f530e3369841ca1ce45c35ed43">2807325</a>: fix: activate IRC keepalive monitoring

Problem:
The client declared ping interval and timeout settings and implemented a ping timeout reconnect path, but authenticated connections never scheduled the ping method.

Impact:
Half-open WebSocket connections could remain marked open while outgoing commands timed out, delaying recovery until the runtime or TCP stack eventually detected abnormal closure.

Cause:
No lifecycle hook started keepalive after IRC authentication, and teardown did not centralize timer cleanup.

Change:
Start one client-originated IRC keepalive interval after numeric 376, prevent overlapping outstanding pings, clear timers on PONG and teardown, and add compatible millisecond constructor options while retaining mutable keepalive fields.

Compatibility:
Existing defaults and public keepalive fields remain available; the new options are additive.

Tests:
Cover authenticated keepalive startup, PONG timeout clearing, missing-PONG reconnect, single outstanding ping behavior, and explicit shutdown cleanup.


Created by <a href="https://github.com/my-badges/my-badges">My Badges</a>