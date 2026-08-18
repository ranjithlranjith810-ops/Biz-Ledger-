# BizLedger — Quick Invoice Party-Specific Rate, Product Count & Old Balance Fix

## Purpose

This change extends the existing Quick Invoice fix with three important business rules:

1. Quick Invoice must remember/load the previous rate for each Party + Product combination.
2. Different customers can have different rates for the same product.
3. Old Balance is a financial adjustment, not a product, so it must not have a quantity and must not be included in the product count.

## 1. Business Requirement — Party-Specific Previous Rate

The same product can have different selling rates for different customers.

Example:

| Product | Customer | Rate |
|---|---|---|
| 2 Door Bero | HI TECH DEVELOPERS | ₹5,500 |
| 2 Door Bero | ABC TRADERS | ₹5,800 |
| 2 Door Bero | XYZ FURNITURES | ₹6,000 |

When creating a Quick Invoice for HI TECH DEVELOPERS and selecting 2 Door Bero, the system should automatically suggest ₹5,500.

When creating one for ABC TRADERS, it should suggest ₹5,800.

**Never use Product alone as the rate-history key.**

The effective pricing key is:

```
Party + Product
```

For branch-specific pricing:

```
Business + Branch + Party + Product
```

## 2. Rate Priority

When Quick Invoice determines a product rate, use this priority:

1. Existing rate on the current invoice item
   ↓
2. Previous Quick Invoice rate for this Party + Product
   ↓
3. Previous Official/Draft invoice rate for this Party + Product
   ↓
4. Current Inventory/default product rate
   ↓
5. 0

When converting an existing Official/Draft invoice, however, the source invoice's historical rate is authoritative and must be preserved.

## 3. Rate History Data Model

Recommended Firebase structure:

```
quickInvoiceRateHistory
  └── businessId
      └── branchId (if branch pricing is separate)
          └── partyId
              └── productKey
                  ├── productName
                  ├── lastRate
                  ├── lastInvoiceId
                  ├── lastInvoiceNo
                  ├── lastDate
                  └── updatedAt
```

Example:

```js
quickInvoiceRateHistory: {
  businessId: {
    party_001: {
      "2-door-bero": {
        productName: "2 Door Bero",
        lastRate: 5500,
        lastInvoiceId: "qin_0032",
        lastInvoiceNo: "QIN/0032",
        lastDate: "2026-08-17",
        updatedAt: 1786940000000
      }
    }
  }
}
```

If the existing application already has a suitable price-history structure, reuse it rather than creating a duplicate Firebase node.

## 4. Product Key

Preferred:

```
productKey = item.productId
```

Fallback:

```
productKey = normalizeProductKey(item.name || item.desc)
```

Example:

```js
function normalizeProductKey(name){
  return String(name || '')
    .trim()
    .toLowerCase()
    .replace(/\s+/g, ' ');
}
```

Do not depend on display-name spelling if a stable Product ID exists.

## 5. Party Key

Preferred:

```
partyKey = party.id
```

Fallback only for legacy data:

```
partyKey = normalizePartyName(party.name)
```

Do not use customer display name as the primary key when a Party ID exists.

## 6. Loading Previous Rate When Party Changes

Flow:

```
Party selected
      ↓
Load that party's rate history
      ↓
Product selected
      ↓
Find Party + Product
      ↓
Apply previous rate
```

Do not overwrite an existing manually entered rate just because the party/product selection changed.

Only auto-populate when:

- a new product is selected;
- the rate is blank/default; or
- the user explicitly requests "Use previous rate".

## 7. Product Selection

Recommended helper:

```js
function getPreviousPartyProductRate(partyId, product){
  if(!partyId || !product) return null;

  var productKey =
    product.id ||
    product.productId ||
    normalizeProductKey(
      product.name || product.desc
    );

  var history =
    _quickRateHistory &&
    _quickRateHistory[partyId];

  if(!history) return null;

  var record = history[productKey];

  if(!record) return null;

  var rate = parseFloat(record.lastRate);

  return Number.isFinite(rate) && rate >= 0
    ? rate
    : null;
}
```

Then:

```js
var previousRate =
  getPreviousPartyProductRate(
    currentPartyId,
    selectedProduct
  );

if(previousRate !== null){
  rateInput.value = previousRate;
}else{
  rateInput.value =
    selectedProduct.rate ||
    selectedProduct.price ||
    0;
}
```

Adapt the field names to the actual BizLedger product model.

## 8. Existing Official/Draft Invoice History

For customers who have never used Quick Invoice, existing invoice history must still provide a previous rate.

Example:

```
HI TECH DEVELOPERS
2 Door Bero

Old Official Invoice → ₹5,200
Newer Official Invoice → ₹5,400
Latest Draft → ₹5,500
```

Use the latest applicable historical rate according to the application's existing invoice-status/business rules.

Search by:

```
Party + Product
```

Never search globally by Product alone.

## 9. Historical Rate Helper

Recommended pattern:

```js
function findPreviousInvoiceRate(partyId, product){
  if(!partyId || !product) return null;

  var productId =
    product.id ||
    product.productId ||
    null;

  var productName =
    normalizeProductKey(
      product.name || product.desc
    );

  var best = null;

  (_invoices || []).forEach(function(inv){

    if(inv.partyId !== partyId){
      return;
    }

    var items = Array.isArray(inv.items)
      ? inv.items
      : [];

    items.forEach(function(item){

      var itemProductId =
        item.productId ||
        item.productID ||
        null;

      var itemName =
        normalizeProductKey(
          item.name || item.desc
        );

      var matches =
        productId && itemProductId
          ? productId === itemProductId
          : itemName === productName;

      if(!matches) return;

      var rate = parseFloat(
        item.price != null
          ? item.price
          : item.rate
      );

      if(!Number.isFinite(rate)){
        return;
      }

      var date =
        inv.invoiceDate ||
        inv.date ||
        '';

      if(!best || date > best.date){
        best = {
          rate: rate,
          date: date,
          invoiceId: inv.id || '',
          invoiceNo:
            inv.invoiceNo ||
            inv.number ||
            ''
        };
      }
    });
  });

  return best;
}
```

The final implementation must use the project's actual invoice field names and existing data/indexing strategy.

## 10. Save Rate History

When a Quick Invoice is successfully saved, update rate history for every real product.

```js
function updateQuickInvoiceRateHistory(invoice){
  if(!invoice || !invoice.partyId) return;

  var partyId = invoice.partyId;
  var items = Array.isArray(invoice.items)
    ? invoice.items
    : [];

  items.forEach(function(item){

    if(item.type === 'old_balance'){
      return;
    }

    var productKey =
      item.productId ||
      normalizeProductKey(
        item.name || item.desc
      );

    if(!productKey) return;

    var rate = parseFloat(
      item.rate != null
        ? item.rate
        : item.price
    );

    if(!Number.isFinite(rate)){
      return;
    }

    // Persist partyId + productKey + rate.
  });
}
```

Only update this history after the invoice is successfully saved.

Canceling an invoice must not change the previous rate.

## 11. Manual Rate Override

Automatic pricing is only a suggestion.

Example:

```
Previous customer rate: ₹5,500

Rate: [ 5500.00 ]

User changes it to:

Rate: [ 5600.00 ]
```

After successful save, the next Quick Invoice for the same Party + Product should suggest ₹5,600.

## 12. Party Change

If the user does:

```
Party A
Product X
→ ₹500

Change Party → Party B
```

the system must resolve:

```
Party B + Product X
```

It must not retain Party A's customer-specific rate.

If Party B has no history:

```
→ Inventory/default product rate
```

## 13. Product Change

If:

```
Party A
Product X → ₹500
```

and the user changes to Product Y, the rate must resolve from:

```
Party A + Product Y
```

It must not retain Product X's rate unless Product Y genuinely has the same saved rate.

Each Quick Invoice row must resolve its own rate independently.

## 14. Multiple Product Rows

Example:

```
Customer: ABC

Product A → ₹825
Product B → ₹1,450
Product C → ₹2,200
```

Changing Product B must not change Product A or Product C.

Do not maintain a single global "previous rate" for the whole invoice.

## 15. Efficient Firebase Reads

Do not query Firebase on every keystroke.

Recommended:

```
Party selected
    ↓
Load party rate history once
    ↓
Cache locally
    ↓
Product selection reads local cache
```

When Party changes:

```
Load new party history
```

When invoice saves:

```
Write latest product rates
```

Recommended runtime cache:

```js
var _quickRateHistory = {};
```

Example:

```js
_quickRateHistory = {
  party_001: {
    "2-door-bero": {
      lastRate: 5500,
      lastDate: "2026-08-17"
    }
  }
};
```

If the project already has a cache/data-loading pattern, integrate with it.

## 16. Quick Invoice Conversion

When converting an Official/Draft invoice:

```
Source invoice rate
        ↓
Quick Invoice rate
```

Example:

```
Official Invoice
Customer: HI TECH DEVELOPERS
Product: 2 Door Bero
Qty: 1
Rate: ₹5,500
```

```
Convert to Quick Invoice

Customer: HI TECH DEVELOPERS
Product: 2 Door Bero
Qty: 1
Rate: ₹5,500
```

Do not replace the source rate with the current Inventory rate.

The source invoice represents the historical transaction.

After saving the new Quick Invoice, that saved rate becomes the latest customer-specific rate.

## 17. Old Balance Is Not a Product

Old Balance must never be represented as:

```js
{
  name: "Old Balance (10/08/2026)",
  qty: 1,
  rate: 50000
}
```

Instead keep it as separate financial data:

```js
{
  oldBalance: {
    amount: 50000,
    date: "2026-08-10"
  }
}
```

Use the project's existing Old Balance data structure if one already exists.

## 18. Old Balance PDF

The PDF should display:

```
#   Product / Item             Qty     Rate        Amount
----------------------------------------------------------
1   2 Door Bero                 1     5,500.00     5,500.00

    Old Balance (10/08/2026)    —        —        50,000.00
----------------------------------------------------------
TOTAL                                             55,500.00

No. of Products: 1
```

The Old Balance row must have:

- Qty  = blank
- Rate = blank
- Amount = balance amount

It must not look like an inventory sale line.

## 19. Product Count

The count must include only actual products.

Recommended:

```js
function isRealQuickInvoiceProduct(item){
  return item &&
    item.type !== 'old_balance' &&
    item.type !== 'balance' &&
    !item.isOldBalance;
}

var productCount =
  items.filter(isRealQuickInvoiceProduct).length;
```

Expected:

```
1 product + Old Balance
→ No. of Products: 1

3 products + Old Balance
→ No. of Products: 3

3 products
→ No. of Products: 3
```

Do not derive the count from PDF row count.

## 20. Do Not Put Old Balance Into Rate History

Old Balance must never create a:

```
Party + Old Balance → Rate
```

record.

It is not a product and must never affect product pricing.

It must also not be included in:

- Product count
- Inventory quantity
- Product sales history
- Customer/product rate history

It only affects the invoice's financial total according to the existing Quick Invoice business rule.

## 21. Complete Rate Resolution

Use this conceptual algorithm:

```js
function resolveQuickInvoiceRate(partyId, item, product){

  // Current item rate
  if(item && item.rate != null){
    var current = parseFloat(item.rate);

    if(Number.isFinite(current) && current > 0){
      return current;
    }
  }

  // Party-specific Quick Invoice history
  var previous =
    getPreviousPartyProductRate(
      partyId,
      product
    );

  if(previous !== null){
    return previous;
  }

  // Existing Official/Draft invoice history
  var historical =
    findPreviousInvoiceRate(
      partyId,
      product
    );

  if(historical){
    return historical.rate;
  }

  // Inventory/default rate
  var inventoryRate = parseFloat(
    product.rate != null
      ? product.rate
      : product.price
  );

  if(Number.isFinite(inventoryRate)){
    return inventoryRate;
  }

  return 0;
}
```

When converting a source invoice, bypass normal resolution and use the source invoice's saved rate.

## 22. Business/Branch Isolation

Because BizLedger supports multiple business/branch contexts, customer-specific rates must not leak between businesses.

Use:

```
Business
+ Branch (if applicable)
+ Party
+ Product
```

as the effective pricing scope.

A customer/product rate belonging to Business A must never appear in Business B.

Use the same Firebase authorization boundary as the existing invoices.

## 23. Existing Data / Migration

Do not require users to manually recreate rate history.

For existing data:

```
Existing invoices
      ↓
Party + Product
      ↓
Find latest historical rate
      ↓
Use immediately
      ↓
Optionally persist to rate-history cache
```

This allows existing BizLedger customers to receive their historical rates immediately.

## 24. Acceptance Tests

### Test 1 — Same product, different customers

Customer A + Product X → ₹500
Customer B + Product X → ₹600

Expected:

```
A → ₹500
B → ₹600
```

### Test 2 — New customer

No previous rate exists.

Inventory rate: ₹750

Expected:

```
₹750
```

### Test 3 — Official invoice history

```
Customer A
Product X
Latest Official Invoice → ₹525

Expected:

Quick Invoice → ₹525
```

### Test 4 — Quick Invoice history wins

```
Official Invoice → ₹525
Latest Quick Invoice → ₹550

Expected:

₹550
```

### Test 5 — Manual override

```
Previous → ₹550
User changes → ₹575
Save

Next invoice:

₹575
```

### Test 6 — Party change

```
Customer A + Product X → ₹500
Change to Customer B

Expected:

Customer B's Product X rate

not ₹500 unless that is actually B's rate.
```

### Test 7 — Product change

```
Customer A
Product X → ₹500
Product Y → ₹900

Selecting Product Y:

₹900
```

### Test 8 — Multiple products

Each row resolves independently.

### Test 9 — Old Balance

```
Product X
Qty 1
Rate ₹5,500

Old Balance ₹50,000

Expected:

No. of Products: 1

Old Balance:
Qty = blank
Rate = blank
Amount = ₹50,000
```

### Test 10 — Official/Draft conversion

Source rate must be copied exactly.

### Test 11 — Original invoice protection

Converting must not modify:

- Original invoice items
- Original quantities
- Original rates
- Original status
- Original invoice number

## 25. Definition of Done

- Quick Invoice identifies the selected Party.
- Previous rates are stored per Party + Product.
- Different customers can have different rates for the same product.
- Product ID is preferred as the product key.
- Party ID is preferred as the party key.
- Existing Official/Draft invoice history supplies legacy rates.
- Latest successfully saved Quick Invoice rate becomes the new previous rate.
- Manual rate overrides are respected.
- Party changes refresh applicable rates.
- Product changes resolve that product's own rate.
- Each product row has independent rate resolution.
- Rate history is business/branch scoped.
- Firebase reads are cached appropriately.
- Source invoice conversion preserves historical rates.
- Old Balance is never treated as a product.
- Old Balance has no quantity.
- Old Balance has no rate.
- Old Balance contributes to the financial total according to existing business rules.
- Old Balance is excluded from product count.
- Old Balance is excluded from product rate history.
- PDF product count contains only actual products.
- Existing invoices can provide historical rates without manual data entry.
- Original invoices are never mutated.

## 26. Final Expected Experience

For HI TECH DEVELOPERS:

```
QUICK INVOICE

Party:
[ HI TECH DEVELOPERS ]

Product:
[ 2 Door Bero ]

Previous rate for this customer:
₹5,500

Qty:
[ 1 ]

Rate:
[ ₹5,500 ]

Amount:
₹5,500
```

For ABC TRADERS:

```
QUICK INVOICE

Party:
[ ABC TRADERS ]

Product:
[ 2 Door Bero ]

Previous rate for this customer:
₹5,800

Qty:
[ 1 ]

Rate:
[ ₹5,800 ]

Amount:
₹5,800
```

With Old Balance:

```
Product / Item                  Qty      Rate       Amount
------------------------------------------------------------
2 Door Bero                      1      5,500        5,500

Old Balance (10/08/2026)         —        —         50,000
------------------------------------------------------------
TOTAL                                              55,500

No. of Products: 1
```

## Core rule

Quick Invoice pricing is customer-specific, product-specific, historical, and manually overridable. Old Balance is financial data, never a product.