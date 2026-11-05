# Why a PWA took 30 seconds to register its service worker

## Symptom

An Angular PWA worked in testing but rarely offered to install, and it didn't load offline for many visitors. The build passed and the service worker file was deployed.

## Cause

The app registered its service worker with:

```ts
provideServiceWorker('ngsw-worker.js', { registrationStrategy: 'registerWhenStable:30000' })
```

That strategy waits for Angular to become stable and uses the number only as a limit. An app with open Firestore listeners and a `setInterval` never becomes stable, so registration always waited the full 30 seconds. Visitors who left sooner never got the service worker.

## Fix

I lowered the limit to 5 seconds. Measured registration time went from 30.2 s to 5.0 s.

## A related trap: testing offline

Loading a route for the first time with no connection reports a failure no real user sees, because the cache only serves what was read before. Load the app online first, then go offline and reload.

One of my offline tests also passed while the browser was showing its own offline page. When a test passes, check that it can fail.
