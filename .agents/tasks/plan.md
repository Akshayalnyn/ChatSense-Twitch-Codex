# Implementation Plan: Convert ChatSense from Node.js to Python/Flask

This plan converts the existing ChatSense Twitch chat analysis app from Node.js/Express to Python/Flask, preserving all business logic, API endpoints, and functionality exactly.

## Framework Decision: Flask

**Rationale:** Flask is chosen over FastAPI because:
1. SSE is simpler in Flask (Response with yield vs. FastAPI's StreamingResponse with async generators)
2. The app is synchronous with background threads (batch timer, WebSocket client, demo timer) rather than async/await throughout
3. Flask's decorator routing matches Express patterns more directly
4. The original server.js uses synchronous patterns with callbacks/timers, not async/await, making Flask a more natural port

## File Structure

**Worktree:** `/Users/akshayalnyn/Documents/Hackathon/Twitch Codex/.worktrees/python-conversion` on branch `python-conversion`

```
.worktrees/python-conversion/
├── app.py                 # Main Flask application (ports server.js)
├── requirements.txt       # Python dependencies
├── .env                   # Environment variables (copied from main workspace)
├── public/                # Frontend (copied from main workspace, no changes)
│   └── index.html
└── test/                  # pytest tests
    ├── test_pipeline.py   # Core logic tests (ports pipeline.test.js)
    └── test_integration.py # Integration tests (Flask routes, SSE)
```

**Dependencies (requirements.txt):**
- Flask==3.0.0 (web framework)
- python-dotenv==1.0.0 (load .env)
- requests==2.31.0 (HTTP client for Twitch API and NVIDIA API)
- websocket-client==1.6.4 (Twitch EventSub WebSocket client)
- pytest==7.4.3 (testing)
- pytest-asyncio==0.21.1 (async test support)

## Technical Approach

### SSE (Server-Sent Events)
Flask streaming response with `Response(stream_with_context(generate()), mimetype='text/event-stream')`. Each client connection gets a `queue.Queue` object added to `session['clients']`. The `broadcast()` function puts messages into all client queues. The SSE generator loops with `queue.get(timeout=30)` to yield messages or keepalive.

### WebSocket Client for Twitch EventSub
Use `websocket-client` library's `WebSocketApp` with callbacks (`on_message`, `on_close`, `on_error`). Run `ws.run_forever()` in a daemon thread started by `threading.Thread(target=ws.run_forever, daemon=True).start()`. The `connectEventSub()` function creates the WebSocket, subscribes to `channel.chat.message` events on `session_welcome`, handles `session_reconnect` for migration, and calls `addMessage()` for incoming chat messages.

### Batch Timer Background Thread
Use `threading.Timer` that recursively schedules itself after calling `processBatch(session)`. The `resetBatchTimer()` function cancels the existing timer and creates a new one with the current `session['config']['interval']` value in milliseconds.

### NVIDIA API Integration
The `analyzeWithNvidia()` function runs in a background thread (started with `threading.Thread(target=analyzeWithNvidia, args=(session, ids), daemon=True).start()`) to avoid blocking Flask requests. It uses `requests.post()` with an 8-second timeout, matching the exact NVIDIA API parameters from server.js: endpoint `https://integrate.api.nvidia.com/v1/chat/completions`, model `nvidia/nemotron-3.5-bolt-e4b`, Bearer auth with `NVIDIA_API_KEY`, `temperature: 0.2`, `response_format: {type: "json_object"}`.

### Demo Mode
The `startDemo()` function uses `threading.Timer` to call `addMessage()` for each scenario in the `scenarios` list (10 tuples of `[id, author, text, viewerId]` from server.js line 23) with 1.05-second intervals. The timer is stored in `session['demoTimer']` and can be cancelled when switching modes.

### OAuth Flow
State stored in a global `oauth_states` dict mapping state string to expiry timestamp. Cookie validation uses `request.cookies.get()` or manual header parsing. Token exchange, refresh, and Twitch API calls use `requests` library with exact headers and retry logic from server.js (retry on 401 after calling `refreshTwitchToken()`).

## Changes to public/index.html

**None.** The frontend is vanilla JavaScript using `EventSource` for SSE and `fetch` for API calls. All API endpoints are preserved exactly, so no frontend changes are required.

## Implementation Steps

- [ ] 1. Create git worktree at `/Users/akshayalnyn/Documents/Hackathon/Twitch Codex/.worktrees/python-conversion` on new branch `python-conversion`. Copy `.env` and `public/` directory from main workspace to worktree.
      Files: worktree directory, .env, public/index.html
      Verify: `cd /Users/akshayalnyn/Documents/Hackathon/Twitch\ Codex/.worktrees/python-conversion && git branch --show-current` outputs `python-conversion`, `.env` and `public/index.html` exist

- [ ] 2. Create `requirements.txt` with Flask, python-dotenv, requests, websocket-client, pytest, pytest-asyncio. Create `app.py` skeleton with Flask app, dotenv load, PORT from env.
      Files: requirements.txt, app.py
      Verify: `cd /Users/akshayalnyn/Documents/Hackathon/Twitch\ Codex/.worktrees/python-conversion && pip install -r requirements.txt` succeeds, `python -c 'import flask; print(flask.__version__)'` prints 3.0.0

- [ ] 3. Port core business logic functions from `server.js` to `app.py`: `new_session()` (line 8-10), `norm()` (line 11), `classify()` (line 12), `scoreMessage()` (line 13), `addMessage()` (line 20), `processBatch()` (line 14-16), `publicState()` (line 17), `broadcast()` stub. Preserve exact logic including regex patterns, scoring formulas, clustering algorithm, normalization.
      Files: app.py
      Verify: `cd /Users/akshayalnyn/Documents/Hackathon/Twitch\ Codex/.worktrees/python-conversion && python -c 'from app import classify, scoreMessage; print(classify("audio is broken")); print(scoreMessage({"text":"what is this?"}, {"activity":"Coding","title":"","category":"","context":"","weights":{"R":0.35,"Q":0.25,"N":0.2,"U":0.2}}, [])["score"] > 0)'` outputs `issue` and `True`

- [ ] 4. Create pytest tests in `test/test_pipeline.py` porting all 7 tests from `test/pipeline.test.js`: (1) stream context changes relevance, (2) question outscores meme, (3) complaints classify as issue, (4) audio reports group by viewers with urgent flag, (5) pinned items survive pruning, (6) weight normalization, (7) NVIDIA fallback (stub for now). Use pytest fixtures for config and sample messages.
      Files: test/test_pipeline.py
      Verify: `cd /Users/akshayalnyn/Documents/Hackathon/Twitch\ Codex/.worktrees/python-conversion && pytest test/test_pipeline.py -v` runs 6 tests pass, 1 xfail (NVIDIA test pending FEAT-002)

- [ ] 5. Implement Flask API endpoints in `app.py`: GET `/api/state` (return publicState), POST `/api/transcript` (update transcript, broadcast), POST `/api/control` (handle pause/resume/interval/weights/context/reset/pin/dismiss/answered/defer/irrelevant actions), POST `/api/mode` (switch demo/live, call startDemo), POST `/api/demo/next` (replay scenarios). Match server.js lines 33-37 exactly.
      Files: app.py
      Verify: `cd /Users/akshayalnyn/Documents/Hackathon/Twitch\ Codex/.worktrees/python-conversion && python app.py &` then `curl http://localhost:3000/api/state | jq .mode` outputs `"demo"`, `curl -X POST http://localhost:3000/api/transcript -H 'Content-Type: application/json' -d '{"text":"test"}' | jq .ok` outputs `true`, kill server

- [ ] 6. Implement SSE endpoint GET `/api/events` using Flask `Response` with `stream_with_context`, Queue-based broadcast. Update `broadcast()` to use `queue.put()` for each client. Update `session['clients']` to store Queue objects. Match server.js line 33.
      Files: app.py
      Verify: `cd /Users/akshayalnyn/Documents/Hackathon/Twitch\ Codex/.worktrees/python-conversion && python app.py &` then `curl http://localhost:3000/api/events` streams `data: {"type":"state",...}\n\n`, kill server

- [ ] 7. Port OAuth flow: implement GET `/auth/twitch` (generate state, set cookie, redirect to Twitch authorize), GET `/auth/twitch/callback` (validate state, exchange code for token, fetch user profile, update session, call connectEventSub stub, redirect). Implement `cookie_value()` helper, global `oauth_states` dict. Match server.js lines 25-27 exactly.
      Files: app.py
      Verify: `cd /Users/akshayalnyn/Documents/Hackathon/Twitch\ Codex/.worktrees/python-conversion && python app.py &` then `curl -I http://localhost:3000/auth/twitch` shows redirect to twitch.tv with Location header, kill server

- [ ] 8. Implement Twitch API helpers: `refreshTwitchToken()` (POST to /oauth2/token with refresh_token grant, validate, update session token), `twitchApi()` (wrapper that retries on 401 after refresh). Match server.js lines 28-29 exactly including error messages and timeout.
      Files: app.py
      Verify: `cd /Users/akshayalnyn/Documents/Hackathon/Twitch\ Codex/.worktrees/python-conversion && python -c 'from app import refreshTwitchToken, twitchApi; print(refreshTwitchToken.__doc__ or "ok"); print(twitchApi.__doc__ or "ok")'` outputs `ok` twice (functions exist)

- [ ] 9. Implement `analyzeWithNvidia()` function: check NVIDIA_API_KEY, handle analyzing state with pendingAnalysisIds, build system/user messages with transcript and candidates, POST to NVIDIA API with exact parameters (model nvidia/nemotron-3.5-bolt-e4b, temperature 0.2, response_format json_object), parse items array, validate and update queue item signals/score/category/reason, handle errors with fallback, recurse for pending IDs. Run in background thread to avoid blocking. Match server.js line 15 exactly.
      Files: app.py
      Verify: Update test/test_pipeline.py test 7 to mock requests.post, test success and failure paths, remove xfail. `cd /Users/akshayalnyn/Documents/Hackathon/Twitch\ Codex/.worktrees/python-conversion && pytest test/test_pipeline.py::test_invalid_nvidia_output -v` passes

- [ ] 10. Port demo mode: implement `scenarios` list (10 tuples from server.js line 23), `startDemo()` function (set session state, create Timer loop calling addMessage with 1.05s delay, cancel after last scenario). Implement `resetBatchTimer()` (cancel existing, create recursive Timer calling processBatch at config interval). Call resetBatchTimer and startDemo at module level (if not test env). Match server.js lines 23-24 and line 39.
      Files: app.py
      Verify: `cd /Users/akshayalnyn/Documents/Hackathon/Twitch\ Codex/.worktrees/python-conversion && python app.py &` then `sleep 5 && curl http://localhost:3000/api/state | jq '.recent | length'` outputs number >0 (demo messages arrived), kill server

- [ ] 11. Implement static file serving: Flask routes for `/` and `/<path:path>` using `send_from_directory('public', ...)` with fallback to `index.html`. Match server.js line 38 catch-all.
      Files: app.py
      Verify: `cd /Users/akshayalnyn/Documents/Hackathon/Twitch\ Codex/.worktrees/python-conversion && python app.py &` then `curl http://localhost:3000/ | grep '<title>ChatSense'` finds title tag, kill server

- [ ] 12. Implement Twitch EventSub WebSocket client: `connectEventSub()` function using websocket-client WebSocketApp, handle session_welcome (subscribe to channel.chat.message via twitchApi POST /helix/eventsub/subscriptions), notification (call addMessage), session_reconnect (migrate to new URL), revocation (disconnect). Run ws.run_forever in daemon thread. Implement `scheduleReconnect()` with exponential backoff. Update /api/mode to close sockets. Update /auth/twitch/callback to call connectEventSub with userId. Match server.js lines 30-32 exactly.
      Files: app.py
      Verify: `cd /Users/akshayalnyn/Documents/Hackathon/Twitch\ Codex/.worktrees/python-conversion && python -c 'from app import connectEventSub, scheduleReconnect; print("ok")'` outputs `ok` (functions exist)

- [ ] 13. Create integration tests in `test/test_integration.py`: test Flask routes with test_client (GET /api/state, POST /api/control, SSE stream), test demo mode cycle (startDemo, messages, queue), mock OAuth callback flow, mock connectEventSub call. Use Flask fixtures and mocks.
      Files: test/test_integration.py
      Verify: `cd /Users/akshayalnyn/Documents/Hackathon/Twitch\ Codex/.worktrees/python-conversion && pytest test/ -v` shows 10+ tests pass (7 pipeline + 3+ integration)

- [ ] 14. Manual end-to-end testing: start Flask server, open browser to `http://localhost:3000`, verify (a) demo mode loads with messages in Recent chat, (b) SSE connection shows 'Listening for signals', (c) queue displays Live Issue/Best Question/Community Insight cards with scores, (d) clicking Replay demo scenario adds messages, (e) Pin/Dismiss/Answered buttons update queue, (f) adjusting weights in Scoring drawer changes scores, (g) static CSS/HTML loads correctly. Test OAuth flow if credentials available (click Connect Twitch, verify redirect, callback, connectEventSub thread starts). Document any issues.
      Files: none (manual testing)
      Verify: Browser at `http://localhost:3000` shows ChatSense UI with demo messages, queue with 3+ cards, controls functional

- [ ] 15. Final verification: run all tests (`pytest test/ -v`), start server (`python app.py`), test all API endpoints with curl (GET /api/state, GET /api/events SSE stream, POST /api/transcript, POST /api/control pause/resume, POST /api/demo/next, GET / static files), verify demo mode and queue updates. Stop server. Commit all changes: `git add -A && git commit -m 'feat: convert ChatSense from Node.js/Express to Python/Flask'`.
      Files: all files in worktree
      Verify: `cd /Users/akshayalnyn/Documents/Hackathon/Twitch\ Codex/.worktrees/python-conversion && pytest test/ -v` all pass, `git log --oneline -1` shows feat commit, `git diff --name-only HEAD~1` shows app.py requirements.txt test/ .env public/

## Summary

This plan ports ChatSense to Python/Flask while preserving every feature:
- **Core logic** (classify, scoreMessage, processBatch) ported with identical algorithms
- **API endpoints** preserved exactly (8 routes + static files)
- **SSE streaming** using Flask Response with Queue-based broadcast
- **WebSocket client** for Twitch EventSub in daemon thread
- **OAuth flow** with state validation and token refresh
- **NVIDIA API** integration in background thread
- **Demo mode** with Timer-based scenarios
- **Tests** ported from Node.js test suite plus Flask integration tests
- **Frontend** works without any changes

The Python version maintains the same single-file architecture (app.py ≈ server.js), making it easy to compare and verify correctness.
