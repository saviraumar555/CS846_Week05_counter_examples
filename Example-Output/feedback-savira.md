## Counterexample — Guideline 4: Document Purpose and Contract

## 1. Rationale

The original guideline assumes that asking the model to summarize purpose and contract only is sufficient to produce safe and accurate function summaries.

However, in infrastructure-style code (e.g., session managers, background services), the boundary between observable behavior and inferred expectations is subtle. The model may unintentionally present assumptions as guarantees, especially when side effects and concurrency are involved.

When this happens, developers may rely on behavior that the code does not actually promise.

Here I provide a counterexample where Guideline 4: Document Purpose and Contract produces a summary that is structurally correct but semantically weak. I then propose an updated guideline that improves transparency and trust.

A major use case for AI code summarization is understanding shared infrastructure modules. These modules often include global state mutation, background threads, and persistence side effects. If assumptions are not clearly labeled, summaries can mislead developers about safety and guarantees.

⸻

## 2. Example Problem

**Task Description:** Summarize all functions in session_store.py using Guideline 4.

Exanple Used: A Python session management module with background threads, logging, token validation, and file storage.
• Session management
• Background sweeper threads
• Telemetry logging
• Token validation
• Disk persistence

```python
import os
import threading
import time
from crypto_utils import sign, constant_time_equal
from telemetry import record_event, inc

_SESSIONS = {}          # session_id -> dict
_LOCK = threading.Lock()
_SWEEPER = None

def start_sweeper(interval_seconds: int = 2) -> None:
    """Start a background thread that removes expired sessions."""
    global _SWEEPER
    if _SWEEPER is not None:
        return

    def loop():
        while True:
            time.sleep(interval_seconds)
            now = time.time()
            with _LOCK:
                expired = [sid for sid, s in _SESSIONS.items() if s.get("expires_at", 0) <= now]
                for sid in expired:
                    _SESSIONS.pop(sid, None)
                    record_event("session.expired", {"sid": sid})

    _SWEEPER = threading.Thread(target=loop, daemon=True)
    _SWEEPER.start()

def create_session(session_id: str, user_id: str, secret: bytes, ttl_seconds: int = 10) -> str:
    """
    Create a session record and return an auth token.
    Token format: "<session_id>.<signature>"
    """
    token = f"{session_id}"
    sig = sign(token, secret)
    with _LOCK:
        _SESSIONS[session_id] = {
            "user_id": user_id,
            "created_at": time.time(),
            "expires_at": time.time() + ttl_seconds,
        }
    inc("writes")
    record_event("session.created", {"sid": session_id, "user": user_id})
    return f"{token}.{sig}"

def validate_token(token_with_sig: str, secret: bytes) -> str | None:
    """
    Validate token signature and expiration.
    Return user_id if valid else None.
    """
    if "." not in token_with_sig:
        inc("fails")
        return None

    token, sig = token_with_sig.split(".", 1)
    expected = sign(token, secret)
    if not constant_time_equal(sig, expected):
        inc("fails")
        record_event("session.invalid_sig", {"sid": token})
        return None

    with _LOCK:
        s = _SESSIONS.get(token)
        if not s:
            inc("fails")
            return None
        if s["expires_at"] <= time.time():
            inc("fails")
            record_event("session.expired_seen", {"sid": token})
            return None
        inc("reads")
        return s["user_id"]

def save_to_disk(path: str) -> None:
    """Persist sessions to disk as JSON."""
    with _LOCK:
        data = dict(_SESSIONS)
    with open(path, "w", encoding="utf-8") as f:
        json.dump(data, f)
    record_event("session.saved", {"path": path})

def load_from_disk(path: str) -> None:
    """Load sessions from disk JSON, overwriting in-memory store."""
    if not os.path.exists(path):
        record_event("session.load_missing", {"path": path})
        return
    with open(path, "r", encoding="utf-8") as f:
        data = json.load(f)
    with _LOCK:
        _SESSIONS.clear()
        _SESSIONS.update(data)
    record_event("session.loaded", {"path": path})
```

## 3. Original Guideline Applied

Original Guideline 4:
Document PURPOSE and CONTRACT, not implementation.

**Prompt Given to GitHub Copilot:**
You are performing code summarization. Follow Guideline 4 strictly: document PURPOSE and CONTRACT, not implementation.
For each function in session_store.py, document purpose, inputs/outputs, preconditions, side effects, and usage.
Do not describe loops or internal steps. Use UNKNOWN if unclear.

**Copilot Output (Excerpt):**
• start_sweeper: Starts a daemon thread; mutates session store; emits telemetry events.
• create_session: Writes to session store; increments telemetry counters.
• validate_token: Reads store and emits telemetry events.

⸻

## 4. Why This Is Weak

Although the output follows the requested structure, it has several issues:
• It presents inferred behavior as factual guarantees
• It does not clearly distinguish assumptions from verified behavior
• It lacks explicit uncertainty labeling
• It blends observable contract with interpretation

This reduces clarity and developer trust.

⸻

## 5. Updated Guideline 4

**Guideline 4 (Improved): Evidence-Based Function Contracts**

When summarizing functions:
• Separate verified behavior from assumptions
• Label inferred expectations explicitly as ASSUMPTION
• Use UNKNOWN when behavior cannot be verified
• Explicitly document visible side effects
• Focus strictly on observable contracts

**Improved Prompt:**

Summarize session_store.py functions using evidence only.
Label inferred behavior as ASSUMPTION.
Use UNKNOWN when unclear.
Describe only observable contracts and side effects.

⸻

## 6. Updated Output Result
Function: start_sweeper
Starts a background thread that removes expired sessions.
Side effects: mutates \_SESSIONS, emits telemetry events, starts a daemon thread.
ASSUMPTION: Intended to be called once at startup.
UNKNOWN: Whether sweep interval is configurable beyond the parameter.
Function: create_session
Creates a session record and returns a signed token.
Side effects: writes to \_SESSIONS, increments counters, records events.
ASSUMPTION: Duplicate session IDs overwrite old sessions.
UNKNOWN: Constraints on TTL values.
Function: validate_token
Validates token signature and expiration and returns user ID if valid.
Side effects: reads \_SESSIONS, emits failure events and counters.
ASSUMPTION: Token format matches create_session output.
UNKNOWN: Whether failure types are externally distinguishable.
Function: save_to_disk
Writes session store to disk as JSON.
Side effects: file I/O and telemetry logging.
ASSUMPTION: Existing files are overwritten.
UNKNOWN: Handling of serialization edge cases.
Function: load_from_disk
Loads session data from disk and replaces in-memory store.
Side effects: file I/O and telemetry logging.
ASSUMPTION: JSON structure matches expected format.
UNKNOWN: Behavior on malformed JSON.

## Problem B — Counterexample to Guideline 6 (Repository-Level Planning)

## 1. Rationale

Guideline 6 asks the model to generate a high-level repository plan. When repository context is incomplete, however, models may invent architecture or relationships that are not grounded in the code.

This creates hallucinated structure and reduces reliability.

⸻

## 2. Example Problem

Task Description: Generate a repository-level plan describing how session_store.py interacts with the repository.

ExampleUsed: A small multi-file Python repository including session_store.py, crypto_utils.py, telemetry.py, and app.py, representing a minimal session management system and its supporting utilities.

## 3. Original Guideline Applied

**Original Guideline 6:**
Develop a global plan for the entire repository.

**Prompt Given to GitHub Copilot:**

Create a high-level repository plan explaining the role of session_store.py, dependencies, callers, and data flow.
Do not guess. Use UNKNOWN if unclear.

**Copilot Output (Excerpt):**

session_store.py is used by app.py and manages session lifecycle.

⸻

## 4. Why This Is Weak

The output:
• Assumes architectural intent
• Does not justify claims with evidence
• Fails to cite where relationships are observed
• Risks hallucinating repository structure

Even when partially correct, the lack of explicit evidence reduces trust.

⸻

## 5. Updated Guideline 6

**Guideline 6 (Improved): Evidence-Only Repository Planning**

When creating repository plans:
• Describe only relationships supported by visible code
• Never infer architecture beyond explicit imports
• Use UNKNOWN for missing information
• Cite evidence from files when possible

**Improved Prompt:**

Generate an evidence-based repository plan.
Cite observable relationships only.
Use UNKNOWN where evidence is missing.

⸻

## 6. Updated Output Result
Repository contains Python modules:
• app.py imports and calls functions from session_store.py
• session_store.py imports from crypto_utils.py and telemetry.py

Role of session_store.py:
Provides session creation, validation, and persistence utilities used by app.py.

Data flow:
app.py → session_store.py → crypto_utils.py / telemetry.py → disk

```

```
