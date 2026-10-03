# Incident 05 — OPNsense web session invalidated after VM reboot

## Category
Application / session

## Context

Creating the `Block Users → IT` firewall rule through the OPNsense web interface.

## Symptom

The "Edit rule" form is filled in correctly (Interface, Action, Source, Destination all valid), but clicking **Save** visibly does nothing — no popup closing, no error message, no new rule appearing in the list.

## Hypotheses tested

1. Missing required field (Interface not selected) → checked and fixed initially, but the issue resurfaced on a similar attempt later.
2. Silent form error, not shown in the UI.

## Diagnosis

The `OPNsense-FW` VM had been **powered off and back on** between two working sessions. The web session (authentication cookie/token for the API) was no longer valid after that reboot, without the web interface showing any clear sign of being logged out or the session having expired — the form kept displaying normally, giving the impression everything worked, while the underlying API calls were silently failing.

## Fix

Fully reloaded the page (F5) — which forced a fresh login and restored a valid session.

## Verification

After reloading, a new attempt to create the rule saved immediately, with the rule visible in the list.

## Lesson learned

After any reboot of the OPNsense VM, always reload the web page before resuming a configuration session — a silently expired session can waste diagnostic time on a problem that doesn't actually exist on the configuration side. A "the click does nothing, with no visible error" behavior should always prompt checking session/connection state before looking for a more complex cause.
