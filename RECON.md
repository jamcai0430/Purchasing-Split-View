# SellToner Recon — one paste, all four answers

Why you're doing this by hand: SellToner's pricing is behind your Google login. I have no browser bridge and no access to your machine in this session, so I can't reach it. These scripts do the looking for you — you paste, then paste the output back to me.

**Safety:** Script A does exactly what your existing scraper already does — one `itemQuery` and one `easyAdd()` for a single SKU (CE255X). It adds no load and nothing SellToner hasn't already seen from you a thousand times. Scripts B and C are read-only.

## Setup

1. Chrome → `https://app.selltoner.com/#/vendor-offer/create`
2. Confirm you're logged in (if it bounces to `sso.selltoner.com`, log in first).
3. F12 → Console tab.
4. If Chrome asks you to type `allow pasting`, do that once.

## Script A — answers Q1 (client-side or POST?) and Q3 (auth token)

Paste, hit enter, wait ~15 seconds for `=== REPORT ===`.

```js
(async () => {
  const R = { ts: new Date().toISOString(), net: [], storage: {}, cookies: null, notes: [] };

  // ---- instrument all network traffic ----
  const origFetch = window.fetch;
  window.fetch = async function (...a) {
    const rec = { via: 'fetch', method: (a[1] && a[1].method) || 'GET', url: String(a[0]),
                  body: a[1] && a[1].body ? String(a[1].body).slice(0, 1500) : null };
    try {
      const r = await origFetch.apply(this, a);
      rec.status = r.status;
      try { rec.resp = (await r.clone().text()).slice(0, 2000); } catch (e) {}
      R.net.push(rec); return r;
    } catch (e) { rec.error = String(e); R.net.push(rec); throw e; }
  };
  const oOpen = XMLHttpRequest.prototype.open, oSend = XMLHttpRequest.prototype.send;
  XMLHttpRequest.prototype.open = function (m, u, ...r) {
    this.__rec = { via: 'xhr', method: m, url: String(u) }; return oOpen.call(this, m, u, ...r);
  };
  XMLHttpRequest.prototype.send = function (b) {
    const rec = this.__rec || {};
    rec.body = b ? String(b).slice(0, 1500) : null;
    rec.reqHeaders = this.__hdrs || null;
    this.addEventListener('loadend', () => {
      rec.status = this.status;
      try { rec.resp = String(this.responseText).slice(0, 2000); } catch (e) {}
      try { rec.respHeaders = this.getAllResponseHeaders(); } catch (e) {}
      R.net.push(rec);
    });
    return oSend.call(this, b);
  };
  const oSetH = XMLHttpRequest.prototype.setRequestHeader;
  XMLHttpRequest.prototype.setRequestHeader = function (k, v) {
    this.__hdrs = this.__hdrs || {};
    this.__hdrs[k] = /auth|token|bearer/i.test(k) ? String(v).slice(0, 60) + '…[truncated]' : v;
    return oSetH.call(this, k, v);
  };

  const mark = R.net.length;

  // ---- find the Angular scope that owns easyAdd() ----
  let scope = null;
  try {
    const walk = (s) => {
      if (!s) return null;
      if (typeof s.easyAdd === 'function') return s;
      return walk(s.$$childHead) || walk(s.$$nextSibling);
    };
    scope = walk(angular.element(document.body).scope());
  } catch (e) { R.notes.push('angular scope walk failed: ' + e); }
  if (!scope) { R.notes.push('NO easyAdd SCOPE FOUND — are you on #/vendor-offer/create and logged in?'); }

  // ---- one itemQuery + one easyAdd, exactly like the existing scraper ----
  let item = null;
  try {
    const r = await fetch('/api/v1/itemQuery?query=CE255X', { credentials: 'include' });
    const j = await r.json();
    const list = Array.isArray(j) ? j : (j.results || j.items || j.data || []);
    item = list.find(x => (x.partNumber || '').toUpperCase() === 'CE255X') || list[0];
    R.itemQuerySample = item;                       // <-- Q4 lives in here
    R.itemQueryFields = item ? Object.keys(item) : null;
    R.itemQueryRespHeaders = [...r.headers.entries()];
  } catch (e) { R.notes.push('itemQuery failed: ' + e); }

  const netBeforeEasyAdd = R.net.length;

  if (scope && item) {
    try {
      scope.itemToAdd = scope.itemToAdd || {};
      scope.itemToAdd.item = item;
      scope.itemToAdd.vendorOfferItem = { boxStyle: 'New Box', boxCondition: 'Pristine', qty: 1 };
      scope.easyAdd();
      await new Promise(res => setTimeout(res, 6000));
      const f = scope.vendorOfferItemFactories || [];
      R.easyAddPrice = f.length ? f[f.length - 1].vendorOfferItem.price : null;
      R.easyAddItemCount = f.length;
      // does the created item carry a server id?
      R.easyAddLastItem = f.length ? JSON.parse(JSON.stringify(f[f.length - 1].vendorOfferItem)) : null;
    } catch (e) { R.notes.push('easyAdd failed: ' + e); }
  }

  R.netDuringEasyAdd = R.net.slice(netBeforeEasyAdd);

  // ---- Q3: auth token ----
  const decodeJwt = (v) => {
    try {
      const p = String(v).split('.');
      if (p.length !== 3) return null;
      const b = JSON.parse(atob(p[1].replace(/-/g, '+').replace(/_/g, '/')));
      return { exp: b.exp, iat: b.iat, ttlMinutes: b.exp && b.iat ? Math.round((b.exp - b.iat) / 60) : null,
               expiresInMinutes: b.exp ? Math.round((b.exp * 1000 - Date.now()) / 60000) : null,
               claimKeys: Object.keys(b) };
    } catch (e) { return null; }
  };
  for (const store of ['localStorage', 'sessionStorage']) {
    R.storage[store] = {};
    try {
      for (let i = 0; i < window[store].length; i++) {
        const k = window[store].key(i), v = window[store].getItem(i) ?? window[store].getItem(k);
        const jwt = decodeJwt(v);
        R.storage[store][k] = jwt
          ? { LOOKS_LIKE_JWT: true, ...jwt }
          : { len: (v || '').length, preview: String(v).slice(0, 80) };
      }
    } catch (e) { R.storage[store] = 'unreadable: ' + e; }
  }
  R.cookies = document.cookie || '(none readable — likely httpOnly, which is fine/good)';

  console.log('=== REPORT ===');
  console.log(JSON.stringify(R, null, 2));
  try { copy(JSON.stringify(R, null, 2)); console.log('(copied to clipboard)'); } catch (e) {}
})();
```

Then: the report is on your clipboard. Paste it to me.

### Reading it yourself, if you want

| What you see in `netDuringEasyAdd` | Means | Consequence |
| --- | --- | --- |
| Empty array | `easyAdd()` computed the price client-side | Best case. One cheap GET per SKU, no server writes, Q2 is moot, traffic halves |
| A POST | Price comes from the server | That endpoint + body is your direct API call. Now run Script B |
| A GET only | Server prices but doesn't persist | Also fine — safe to call repeatedly |

In `easyAddLastItem`, an `id` / `_id` / `vendorOfferItemId` field with a real value is a strong hint the row was persisted server-side. Script B confirms it properly.

## Script B — answers Q2 (does it create server-side records?)

Only run this if Script A showed a POST. This is the question that can invalidate the whole plan, so it's worth doing properly.

The definitive test is whether the item survives a reload — client-side state doesn't, database rows do.

1. After Script A, note `easyAddItemCount` from the report.
2. Hard reload the page (Cmd+Shift+R).
3. Paste this:

```js
(() => {
  const walk = (s) => !s ? null : (typeof s.easyAdd === 'function' ? s : (walk(s.$$childHead) || walk(s.$$nextSibling)));
  const scope = walk(angular.element(document.body).scope());
  const f = (scope && scope.vendorOfferItemFactories) || [];
  console.log('=== PERSISTENCE ===');
  console.log('items present after reload:', f.length);
  console.log(f.length > 0
    ? '⚠️  PERSISTED SERVER-SIDE — every price check writes a row into their DB.'
    : '✅ NOT persisted — client-side/session only. Safe to poll.');
  console.log(f.map(x => x.vendorOfferItem));
})();
```

If it says PERSISTED — that's the finding that changes everything. 7 sweeps/day × ~1,259 SKUs would leave ~9,000 junk draft rows a day in their admin. Highly visible, and the fastest way to lose the account. Response is one of:

1. Replicate the pricing formula offline from the `itemQuery` payload (check whether `itemQuerySample` already contains the base price and multipliers — if so, `easyAdd` may just be doing arithmetic you can copy).
2. Find and call a delete/reset endpoint after each batch (watch the network tab while you clear an offer manually).
3. Drop to the nightly sweep only and accept 24h staleness — which would mean your 4-hour SLA isn't achievable through this path at all.

Also check the Offers section of your SellToner account right now for stray draft offers from past scraper runs. If there's a pile of them, you have your answer already, and some cleanup to do.

## Script C — answers Q4 (cheap change signal)

Read-only. Run any time.

```js
(async () => {
  const out = { headers: {}, probes: [], timestampFields: null };

  // response headers on itemQuery — ETag/Last-Modified would let us diff cheaply
  try {
    const r = await fetch('/api/v1/itemQuery?query=CE255X', { credentials: 'include' });
    out.headers = Object.fromEntries(r.headers.entries());
    const j = await r.json();
    const list = Array.isArray(j) ? j : (j.results || j.items || j.data || []);
    const it = list[0] || {};
    out.timestampFields = Object.keys(it).filter(k =>
      /updat|modif|version|revis|changed|timestamp|date|price|cost|rate|tier/i.test(k));
    out.sampleValues = Object.fromEntries(out.timestampFields.map(k => [k, it[k]]));
  } catch (e) { out.headerErr = String(e); }

  // probe for a bulk catalog endpoint — the jackpot find
  const candidates = [
    '/api/v1/items', '/api/v1/items?page=1', '/api/v1/items?limit=5',
    '/api/v1/itemList', '/api/v1/catalog', '/api/v1/priceList', '/api/v1/prices',
    '/api/v1/itemQuery?query=', '/api/v1/item', '/api/v1/products'
  ];
  for (const u of candidates) {
    try {
      const r = await fetch(u, { credentials: 'include' });
      const t = (await r.text()).slice(0, 300);
      out.probes.push({ url: u, status: r.status, preview: t });
    } catch (e) { out.probes.push({ url: u, error: String(e) }); }
    await new Promise(z => setTimeout(z, 400)); // be polite
  }

  console.log('=== CHANGE SIGNAL ===');
  console.log(JSON.stringify(out, null, 2));
  try { copy(JSON.stringify(out, null, 2)); console.log('(copied)'); } catch (e) {}
})();
```

What would be a big deal:

- An `updatedAt` / `version` field on items → poll that instead of pricing every SKU.
- `ETag` or `Last-Modified` headers → conditional requests, near-zero cost.
- Any probe returning 200 with a list of items → one paginated sweep replaces 1,259 individual calls. This is the single highest-value outcome in the whole recon, and would make your 4-hour SLA nearly free.

Note: I guessed those endpoint paths. If Script A's net log shows other `/api/v1/...` routes the app calls on its own, add them to the candidate list — the app tells you its own API surface for free.

## What to send back

Paste me the three report blobs. I'll turn them into the concrete collector design — real endpoints, real payloads, real token lifetime — instead of the placeholders currently in the plan.

If Script A errors out, send the error text; the most likely cause is the Angular scope walk failing because the page structure shifted, which is a two-minute fix.
