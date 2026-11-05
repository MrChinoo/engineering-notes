# A 500 on preflight is not a CORS problem

**October 2026. 49 Cloud Functions gen2 across 6 projects.**

## Symptom

An admin panel showed `internal`, and the browser console showed a CORS error. The CORS configuration hadn't changed. Every public gen2 function was failing, on every project.

## Diagnosis

I sent the preflight by hand:

```bash
curl -i -X OPTIONS <function-url> -H "Origin: https://example.com" -H "Access-Control-Request-Method: POST"
```

It returned a 500 from Google, before any of my code ran. The browser reports a failed preflight as a CORS error, so the console message was misleading.

The logs said the instance could not start and the container failed to listen on the port. There was no output from the container itself.

## Cause

Gen2 functions receive automatic base image updates. A new `nodejs22` image had just been published, and containers built on it never started listening on port 8080.

Two things made it worse:

- Rebuilding on `nodejs22` kept failing.
- A cleanup rule in Artifact Registry deleted images older than one day, so there was no image left to go back to.

## Fix

1. I tested one function on `nodejs24` as a control. It started on the first try.
2. I rebuilt the rest from the source already deployed, without using local code. Local code could have published unreleased changes.
3. I added a retention rule that keeps the most recent images.
4. I added a monitor that sends a preflight to critical functions every 5 minutes and alerts after two consecutive 500s.

## What I took from it

- Read a CORS error as "the preflight failed", then check the status code.
- Change one function first, as a control, before touching all of them.
- Default cleanup rules can remove the image you would need for a rollback.

The tool from this incident: [gcf-fleet-doctor](https://github.com/MrChinoo/gcf-fleet-doctor).
