# Deploy folder — drop-in replacement

`index.html` here is repo HEAD with all the gaps closed. Copy this whole folder's contents to the
repo root (you still need to supply the three icon PNGs — a red 100 mark).

What changed vs repo HEAD:

- `enableIndexedDbPersistence` — offline logs queue instead of vanishing, and a cold offline launch
  reads from cache. `saveShared()` also mirrors to localStorage every write as a second belt.
- Sync status line on the Board: live / offline / unavailable, with the last write time.
- Board scoped by URL hash (`#ourgroup`). **No hash keeps the existing doc**, so your current board
  and everyone's data are untouched — new groups get their own.
- `sw.js` + `manifest.webmanifest` + update toast, and a "Version 1.1.0 · Check for update" row on
  the Board so a stuck app is one tap from current.
- The red 100 on Today is now the log button itself; stat row (week / streak / rank) under it; the
  sticky slab only appears on Board and Group.
- Sparkline of your last 8 weeks, built from real `state.history`.
- Haptic on log; undo moved to a dark bar above the tabs.
- Daily reminder: pick a time, "Add to calendar" downloads a repeating .ics with a 0-minute alarm.
  iOS opens the add-to-calendar sheet from the download, so it needs no notification permission and
  survives the app being closed.
- Logs are locked to the real current day. `state.lastSeen` only moves forward, so winding the phone
  clock back to fill in a missed day is refused with "That day is closed."
- Icons included: icon-180/192/512.png — the red 100 mark.

## Firestore rules

**Do this in order or you will lock everyone out.**

1. **Firebase console → Authentication → Sign-in method → enable Anonymous.** Nothing works
   without this; the app signs in silently on launch.
2. **Deploy this build** (v2.2.0). It signs in anonymously and waits for that before its first
   read. The current permissive rule still allows it, so nothing breaks while people update.
3. **Give everyone a day to open the app once.** Their device gets a stable anonymous uid.
4. **Then** tighten the rule:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /pushup/{docId} {
      allow read, write: if request.auth != null
        && request.resource.data.value is string
        && request.resource.data.value.size() < 100000;
    }
  }
}
```

That closes the world-writable hole: a random visitor can no longer wipe the board, and the size
cap stops anyone bloating the document. Anonymous uids persist per device, so nobody has to log in.

If someone gets stuck after step 4, they were still on an old build — have them open the site in
Safari once (not the home-screen icon) to pick up the new version.


Everything below is the reference for what the service worker does and how to deploy it.

---

# Making the home-screen shortcut updatable

Why it's stuck today: `index.html` is the only file in the repo. A home-screen web clip has no
address bar and no pull-to-refresh, so iOS serves it from the HTTP cache with no way to force a
reload. A service worker is what gives you a version you control.

## 1. Add these files next to index.html

- `sw.js` — network-first for the page, cache-first for fonts/icons.
- `manifest.webmanifest` — makes it a real standalone app (also fixes the icon and theme colour).
- `icon-180.png`, `icon-192.png`, `icon-512.png` — your own icons (the red 100 mark).

## 2. In index.html `<head>`

```html
<link rel="manifest" href="manifest.webmanifest">
<link rel="apple-touch-icon" href="icon-180.png">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-title" content="100">
```

## 3. At the end of index.html, before `</body>`

```html
<script>
const APP_VERSION = '1.0.0';   // bump this every deploy
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    navigator.serviceWorker.register('sw.js').then(reg => {
      if (reg.waiting && navigator.serviceWorker.controller) showUpdateToast(reg.waiting);
      reg.addEventListener('updatefound', () => {
        const w = reg.installing;
        if (!w) return;
        w.addEventListener('statechange', () => {
          if (w.state === 'installed' && navigator.serviceWorker.controller) showUpdateToast(w);
        });
      });
      // Safari never fires updatefound on its own for a backgrounded app — poke it on return.
      document.addEventListener('visibilitychange', () => { if (!document.hidden) reg.update().catch(()=>{}); });
      window.checkForUpdate = () => reg.update().catch(()=>{});
    }).catch(()=>{});

    let reloaded = false;
    navigator.serviceWorker.addEventListener('controllerchange', () => {
      if (reloaded) return; reloaded = true; location.reload();
    });
  });
}
function showUpdateToast(worker) {
  if (document.getElementById('updateToast')) return;
  const el = document.createElement('div');
  el.id = 'updateToast';
  el.style.cssText = 'position:fixed;left:0;right:0;bottom:0;z-index:60;background:#201e1d;color:#f3f2f2;'
    + 'padding:14px 20px calc(14px + env(safe-area-inset-bottom));display:grid;'
    + 'grid-template-columns:minmax(0,1fr) auto auto;gap:14px;align-items:center;font-family:Archivo,system-ui,sans-serif';
  el.innerHTML = '<p style="margin:0;font-size:13px;font-weight:600">A new version is ready.</p>'
    + '<button id="upLater" style="border:0;background:none;cursor:pointer;font:inherit;font-size:11px;font-weight:800;'
    + 'letter-spacing:.12em;text-transform:uppercase;color:rgba(243,242,242,.6)">Later</button>'
    + '<button id="upGo" style="border:0;background:#ec3013;color:#fff;cursor:pointer;font:inherit;padding:11px 14px;'
    + 'font-size:11px;font-weight:800;letter-spacing:.12em;text-transform:uppercase">Update</button>';
  document.body.appendChild(el);
  el.querySelector('#upGo').onclick = () => worker.postMessage({ type: 'SKIP_WAITING' });
  el.querySelector('#upLater').onclick = () => el.remove();
}
</script>
```

## 4. Every deploy

Bump **both** `APP_VERSION` and the `CACHE` constant in `sw.js` (`pushups-v1` → `pushups-v2`).
The cache name is what makes the browser treat it as a new worker.

## 5. Escape hatch

Show the version somewhere in the UI with a "Check for update" tap that calls
`window.checkForUpdate()`. Then a stuck app never needs deleting again — worst case, one tap.

Note: GitHub Pages serves HTML with a short max-age, so the *first* load after a deploy may still
come from the HTTP cache. The `visibilitychange` poke plus the manual check cover that.
