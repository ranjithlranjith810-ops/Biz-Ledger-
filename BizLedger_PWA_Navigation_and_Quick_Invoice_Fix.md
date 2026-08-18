# BizLedger --- PWA Navigation, Pull-to-Refresh Protection & Quick Invoice Fix

This change set fixes three related production issues in the BizLedger PWA:

1. Android/iOS PWA back navigation must move page-by-page instead of immediately leaving the app.
2. The PWA must not reload when the user pulls/swipes downward from the top of a page. This must not wipe unsaved UI state or force Firebase data to reload.
3. Convert to Quick Invoice must copy product/item details from the source Official/Sales/Purchase/Draft invoice, not only the customer/vehicle fields.

The existing uploaded build was inspected before defining this change. The current Quick Invoice conversion code explicitly excludes source invoice items, while the normal invoice item model contains `desc`, `hsn`, `unit`, `qty`, and `price`.

## 1. PWA Navigation — Required Behaviour

### Target behaviour

The app should behave like a normal mobile application:

```
Dashboard/Home
   ↓
Invoices
   ↓
Invoice detail
   ↓
Edit invoice
   ↓
Back
   → Invoice detail
   → Invoices
   → Dashboard/Home
```

For normal pages:

```
Dashboard → Invoices → Products → Parties
                         ↑
                    Back button
```

The physical Android back button, browser back gesture, and browser back button must use the same navigation history.

### Important rule

Do not use `history.pushState()` only for overlays and leave page navigation outside browser history.

Every meaningful SPA page transition must create a history state.

## 2. Central Navigation State

Use one consistent history state format:

```js
{
  bl: true,
  type: 'page',
  page: 'd'
}
```

For an overlay:

```js
{
  bl: true,
  type: 'overlay',
  page: 'i',
  overlay: 'ovdetail'
}
```

Initialize the app history once:

```js
function initBLHistory(){
  var current = history.state;
  if(!current || current.bl !== true){
    history.replaceState(
      { bl:true, type:'page', page:'d' },
      '',
      location.href
    );
  }
}
```

Call this once during application startup (`showApp()`).

## 3. Page Navigation

Replace the current page-switch behaviour with a history-aware version.

```js
var _currentPage = 'd';
var _historyApplying = false;

function applyPage(pg){
  _currentPage = pg;
  document.querySelectorAll('.page').forEach(function(p){ p.classList.remove('on'); });
  document.querySelectorAll('.ni').forEach(function(n){ n.classList.remove('on'); });
  var page = g('pg-'+pg);
  if(page) page.classList.add('on');
  var nav = document.querySelector('.ni[onclick*="swPg(\''+pg+'\',"]');
  if(nav) nav.classList.add('on');
  var renderMap={
    'd': function(){ renderD(); },
    'i': function(){ renderI(); },
    'inv': function(){ renderPr(); },
    'p': function(){ renderPa(); },
    'veh': function(){ if(typeof renderVeh==='function') renderVeh(); },
    'pay': function(){ if(typeof renderPay==='function') renderPay(); },
    'emp': function(){ if(typeof renderEmp==='function') renderEmp(); }
  };
  if(renderMap[pg]) setTimeout(renderMap[pg], 0);
}

function swPg(pg, el){
  if(!pg) return;
  if(_currentPage === pg && !_historyApplying) return;
  if(document.querySelector('.ov.on')) closeAllOverlaysDirect();
  if(!_historyApplying){
    history.pushState({ bl:true, type:'page', page:pg }, '', location.href);
  }
  applyPage(pg);
}
```

Result: visiting Dashboard → Invoices → Products → Parties pushes those application states, and Android back returns Parties → Products → Invoices → Dashboard instead of leaving the PWA.

## 4. Overlay / Bottom-Sheet History

```js
function opn(id){
  var el = g(id);
  if(!el) return;
  el.classList.add('on');
  document.body.style.overflow = 'hidden';
  var state = { bl:true, type:'overlay', page:_currentPage, overlay:id };
  if(!_historyApplying) history.pushState(state, '', location.href);
}

function closeOverlayDirect(id){
  var ov = g(id);
  if(ov) ov.classList.remove('on');
  if(!document.querySelector('.ov.on')) document.body.style.overflow = '';
  if(id === 'ovinv' || id === 'ovqinv') _invoiceFormDirty = false;
}

function closeAllOverlaysDirect(){
  document.querySelectorAll('.ov.on').forEach(function(ov){ closeOverlayDirect(ov.id); });
}

function cls(id){
  var ov = g(id);
  if(!ov || !ov.classList.contains('on')) return;
  // If this overlay owns the current history entry, go back so popstate performs the actual close.
  if(history.state &&
     history.state.bl === true &&
     history.state.type === 'overlay' &&
     history.state.overlay === id){
    history.back();
    return;
  }
  closeOverlayDirect(id);
}
```

## 5. Hardware Back / Browser Back / Swipe Back

One `popstate` handler as the single source of truth:

```js
window.addEventListener('popstate', function(e){
  var state = e.state;
  _historyApplying = true;
  if(state && state.bl === true){
    if(state.type === 'overlay'){
      closeAllOverlaysDirect();
      if(state.page && state.page !== _currentPage) applyPage(state.page);
      var ov = g(state.overlay);
      if(ov){ ov.classList.add('on'); document.body.style.overflow = 'hidden'; }
    }else if(state.type === 'page'){
      closeAllOverlaysDirect();
      applyPage(state.page || 'd');
    }else{
      closeAllOverlaysDirect();
      applyPage('d');
    }
  }else{
    closeAllOverlaysDirect();
    applyPage('d');
  }
  _historyApplying = false;
});
```

Back navigation acceptance test — test all of these:
- Android physical back button
- Chrome browser back button
- Android back gesture
- iOS Safari/PWA back gesture where supported
- Back from an invoice editor
- Back from Quick Invoice editor
- Back from detail sheets
- Back through multiple pages
- Back after opening and closing several sheets

Expected: back always removes the newest BizLedger navigation layer first.

## 6. Unsaved Invoice Protection

Keep the existing `beforeunload` protection for real browser/PWA exits. The SPA history system handles normal internal back navigation. Back from an invoice editor closes the editor and returns to the detail sheet. If the editor has unsaved changes, an in-app confirmation should be shown before discarding them.

```js
function confirmCloseInvoiceForm(id){
  if(!_invoiceFormDirty){ closeOverlayDirect(id); return true; }
  var ok = confirm('You have unsaved invoice changes. Leave without saving?');
  if(!ok) return false;
  _invoiceFormDirty = false;
  closeOverlayDirect(id);
  return true;
}
```

## 7. Disable Pull-to-Refresh in the PWA

The problem where the user swipes from the top downward and Chrome reloads the PWA is browser pull-to-refresh. This is separate from the Service Worker.

### CSS protection

```css
html, body { overscroll-behavior-y: none; }
html { overscroll-behavior-x: none; }
body { overscroll-behavior: none; overflow-x: hidden; }
#APP { overscroll-behavior-y: none; overscroll-behavior-x: none; }
.shwrap { overscroll-behavior-y: contain; -webkit-overflow-scrolling: touch; }
```

Important: do not globally set `touch-action: none;` — that would damage normal scrolling.

## 8. Additional Pull-to-Refresh Guard

If Chrome still performs pull-to-refresh, add a controlled touch guard that only blocks a downward drag when the app is already at the top (never disables normal scrolling):

```js
var _ptrStartY = 0;
var _ptrTracking = false;

document.addEventListener('touchstart', function(e){
  if(!e.touches || !e.touches.length) return;
  _ptrStartY = e.touches[0].clientY;
  _ptrTracking = true;
}, {passive:true});

document.addEventListener('touchmove', function(e){
  if(!_ptrTracking || !e.touches || !e.touches.length) return;
  var y = e.touches[0].clientY;
  var dy = y - _ptrStartY;
  if(dy <= 0) return;
  var target = e.target;
  var scrollParent = target.closest && target.closest('.shwrap, .page, #APP');
  if(!scrollParent) return;
  if(scrollParent.scrollTop <= 0) e.preventDefault();
}, {passive:false});

document.addEventListener('touchend', function(){ _ptrTracking = false; }, {passive:true});
document.addEventListener('touchcancel', function(){ _ptrTracking = false; }, {passive:true});
```

Goal: normal vertical scroll works, but at top + swipe downward → no browser pull-to-refresh → no page reload → no Firebase/UI state loss.

## 9. Service Worker Requirements

The current Service Worker uses network-first navigation and does not cache Firebase realtime data. That is correct and should remain. Do not cache `firebaseio.com`, `firebase.googleapis.com`, `identitytoolkit`, or `securetoken`. After changing `index.html`, increment the cache version to force installed PWAs to receive the fixes:

```js
const CACHE = 'bizledger-v8';
```

## 10. Quick Invoice — Current Bug

The current conversion function excludes source invoice items, so converting an Official/Sales/Purchase/Draft invoice creates a new Quick Invoice with customer/vehicle information but an empty product list.

## 11. Quick Invoice Product Mapping

Regular invoice items: `{ desc, hsn, unit, qty, price }`. Quick Invoice items: `{ name, qty, rate, amount }`. Explicitly map:

| Official/Draft Invoice | Quick Invoice |
|---|---|
| `desc` | → | `name` |
| `qty` | → | `qty` |
| `price` | → | `rate` |
| `qty × price` | → | `amount` |
| `unit` | → | optional metadata |
| `hsn` | → | optional metadata |

GST must not be copied into Quick Invoice calculations (Quick Invoice is GST-free).

## 12. Corrected convertToQuickInv()

Converts items first, then opens the Quick Invoice form with the converted items, and fills header fields. The source invoice price always wins over current inventory price (historical transaction price). The original invoice is never mutated.

`cls()` now uses `history.back()` when the sheet owns the current history entry, so the new form must open only after that back-navigation settles — otherwise the pending popstate would close the freshly opened sheet. The form open is therefore deferred 100 ms (same pattern already used by `doEditInv`/`doEditQInv`), then a further 150 ms wait for the form DOM before filling items.

```js
function convertToQuickInv(inv){
  if(!inv){ toast('No invoice selected','r'); return; }
  cls('ovdetail');
  cls('ovqdetail');
  var sourceItems = Array.isArray(inv.items) ? inv.items : [];
  var convertedItems = sourceItems.map(function(it){
    var qty = parseFloat(it.qty != null ? it.qty : it.quantity != null ? it.quantity : 0) || 0;
    var rate = parseFloat(it.price != null ? it.price : it.rate != null ? it.rate : it.unitPrice != null ? it.unitPrice : 0) || 0;
    var amount = parseFloat(it.amount);
    if(!amount || amount <= 0) amount = qty * rate;
    return {
      name: (it.desc || it.name || it.productName || '').trim(),
      qty: qty || 1,
      rate: rate || 0,
      amount: Math.round(amount * 100) / 100,
      unit: it.unit || 'PCS',
      hsn: it.hsn || it.hsnCode || ''
    };
  }).filter(function(it){ return it.name; });

  setTimeout(function(){
    openQInvF(null);
    setTimeout(function(){
      _qitems = convertedItems.length ? convertedItems : [{name:'', qty:1, rate:0, amount:0}];
      buildQItemRows();
    if(g('qidt')) g('qidt').value = inv.invoiceDate || new Date().toISOString().split('T')[0];
    var name = inv.partyName || inv.receiverName || inv.receiver || '';
    if(g('qirecv')) g('qirecv').value = name;
    var party = null;
    if(inv.partyId) party = (_par || []).find(function(p){ return p.id === inv.partyId; });
    if(!party && name) party = (_par || []).find(function(p){ return (p.name || '').toLowerCase() === name.toLowerCase(); });
    if(party){
      fillQParty(party.id);
    }else{
      if(g('qi-state')) g('qi-state').value = inv.state || inv.partyState || '';
      if(g('qi-district')) g('qi-district').value = inv.district || inv.partyDistrict || inv.partyCity || '';
      if(g('qi-statecode')) g('qi-statecode').value = inv.stateCode || inv.partyStateCode || '';
      if(g('qipt')) g('qipt').value = inv.paymentTerms || 'Immediate';
    }
    if(g('qiveh')) g('qiveh').value = inv.vehicleNo || inv.vehicleNumber || inv.vehicle || '';
    if(g('qi-notes')) g('qi-notes').value = inv.notes || '';
    if(inv.oldBalance && inv.oldBalance.amount){
      if(g('qi-ob-toggle')) g('qi-ob-toggle').checked = true;
      if(g('qi-ob-amt')) g('qi-ob-amt').value = inv.oldBalance.amount;
      if(g('qi-ob-date')) g('qi-ob-date').value = inv.oldBalance.date || '';
      toggleQOldBal();
    }
    recalcQTotals();
    toast(convertedItems.length ? convertedItems.length + ' product(s) copied from invoice.' : 'No products found in the source invoice.', convertedItems.length ? 'g' : 'r');
    }, 150);
  }, 100);
}
```

## 13. Preserve Exact Product Details

Do not replace the saved rate with the current Inventory selling price. Conversion priority: source invoice `price` → source `rate` → inventory price only if the source price is missing.

## 14. Product Metadata

Save optional source metadata (`unit`, `hsn`) alongside each item so future PDF/report features can use the original unit/HSN without changing the visible simplified form.

## 15. Official + Draft Invoice Compatibility

Conversion must work for official sales invoices, draft sales invoices, purchase invoices, and any regular invoice stored under the existing invoices node — read from `inv.items`, not the current Inventory table.

## 16. Do Not Mutate the Original Invoice

Always map into a new array; never modify `inv.items`.

## 17. Quick Invoice Acceptance Tests

- **Test A — Official invoice:** INV/25-26/001, customer ABC Traders, Product A (10 × ₹790), Product B (5 × ₹1200) → converted QI shows Product A 10×₹790=₹7900, Product B 5×₹1200=₹6000, total ₹13900.
- **Test B — Draft invoice:** draft with 3 products → all 3 appear automatically.
- **Test C — Historical price:** inventory ₹900, invoice ₹790 → QI rate remains ₹790.
- **Test D — Empty item:** ignore empty/invalid item rows.
- **Test E — No products:** show a clear message that no source products were found.
- **Test F — Original unchanged:** invoice number, status, quantities, rates, items all unchanged.

## 18. Final Production Checklist

### Navigation
- [ ] Initialize a BizLedger root history state
- [ ] Every SPA page change uses pushState
- [ ] Overlay open uses pushState
- [ ] Overlay close uses history.back() when its state is active
- [ ] popstate is the single source of truth
- [ ] Android hardware back tested
- [ ] Android gesture back tested
- [ ] Chrome back tested
- [ ] Back sequence ends at Dashboard/Home
- [ ] Unsaved invoice changes protected
- [ ] No broken/duplicate history entries after opening/closing sheets

### Pull-to-refresh
- [ ] `overscroll-behavior: none` on the document
- [ ] Appropriate overscroll rules on app/sheet scroll containers
- [ ] Test Android Chrome installed PWA
- [ ] Normal scrolling works
- [ ] Downward swipe at page top → no reload
- [ ] Firebase/UI data not lost
- [ ] Only use the JS touch guard if CSS alone is insufficient

### Quick Invoice
- [ ] Official invoice conversion copies products
- [ ] Draft invoice conversion copies products
- [ ] Purchase invoice conversion copies products
- [ ] `desc → name`
- [ ] `qty → qty`
- [ ] `price/rate → rate`
- [ ] Amount preserved/calculated
- [ ] Unit/HSN metadata preserved
- [ ] Historical invoice rate wins over current inventory price
- [ ] GST not introduced into Quick Invoice
- [ ] Original invoice unchanged
- [ ] Quick Invoice total recalculates correctly
- [ ] Quick Invoice product count correct

## 19. Definition of Done

The fix is complete only when a mobile user can:
- Open BizLedger, navigate through several pages, press Back repeatedly, move back one screen at a time, and reach Dashboard/Home without unexpectedly leaving the app.
- Open an invoice editor, swipe downward at the top without triggering browser refresh, and keep all entered UI data intact.
- Convert an Official invoice to Quick Invoice and see every original product, quantity, and historical rate automatically populated.
- Convert a Draft invoice the same way, then save the Quick Invoice without modifying the original invoice.

This is a navigation/state-management fix plus a Quick Invoice data-mapping fix — not a workaround that reloads or reconstructs application data after every gesture.