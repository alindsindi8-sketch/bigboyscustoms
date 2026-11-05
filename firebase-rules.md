# BIGBOYS CUSTOMS — Firebase Security Rules

**Firestore** rules for this site, plus the setup they assume. Publish these in the
Firebase console. Product photos are stored in Firestore; Firebase Storage is not used.
The rules themselves do not need a service-account key. Booking email notifications are
sent through EmailJS from the browser as described in `README.md`.

---

## 1. What the site actually reads and writes

Knowing the exact calls makes the rules reviewable instead of guessed.

| Collection | Who touches it | Operation in the code |
| --- | --- | --- |
| `products` | Every visitor | Public catalogue list; product details, prices, and compressed image data |
| `products` | Admin only | create, update, delete (`/#admin` product form); exact stock is forbidden here |
| `categories` | Every visitor | Public category list for shop filters |
| `categories` | Admin only | create and update shop categories from the dashboard |
| `inventory` | Admin only | read and update wheel stock; stock counts are not exposed in public product documents |
| `orders` | Anyone (no login needed) | **create only** — a booking request, always `status: 'pending'` |
| `orders` | Admin only | list and read private customer/vehicle details; update status or delete |
| `admins/{uid}` | Staff UI | read **own** doc only, to resolve a role when custom claims are not set |
| `admins/{uid}` | Owner only | create, update, delete staff role documents |
| `products.imageData` | Every visitor | compressed JPEG data URL stored in the public product document |

Two things the code **never** does, and the rules therefore forbid outright:

* a visitor writing or changing the price, status or totals of anything;
* anyone but staff listing customers, orders or messages (that would leak names,
  phone numbers and addresses to the internet).

---

## 2. How "am I an admin?" is decided

The frontend check in `assets/js/auth.js` (`BB.requireAdmin`) is **only** for hiding
menu buttons and showing the login screen. Hiding a button is not security: anyone can
run a fetch from devtools. The enforcement lives in the rules below.

A request counts as staff when **either** is true:

1. **Custom claim** on the ID token: `{"admin": true}` (or `{"role": "admin"}`).
   Firebase Console → *Authentication* → *Users* → click the user → **Edit claims**:
   ```json
   { "admin": true }
   ```
   This is the recommended way — the rules resolve without an extra document read,
   and the claim cannot be edited from the browser.
2. **Role document** `admins/{uid}` with `role` one of `owner`, `admin`, `staff`.
   This is the no-backend fallback the dashboard also uses for role wording, and it is
   how you add the first admin before any claims exist (see §6).

`owner` is the only role that can change the `admins` collection, so a staff login can
never promote itself.

---

## 3. Firestore Security Rules

Publishable as-is. Paste into
*Firebase Console → Firestore Database → Rules* and press **Publish**.

```
rules_version = '2';

service cloud.firestore {
  match /databases/{database}/documents {

    /* ================= helpers ================= */

    function signedIn() {
      return request.auth != null;
    }

    // 1. custom claim, set in the Firebase console (no document read needed)
    function hasAdminClaim() {
      return signedIn() &&
        (request.auth.token.admin == true || request.auth.token.role == 'admin');
    }

    // 2. protected role document, admins/{uid}
    function hasAdminDoc() {
      return signedIn() &&
        exists(/databases/$(database)/documents/admins/$(request.auth.uid));
    }

    function adminRole() {
      return get(/databases/$(database)/documents/admins/$(request.auth.uid))
        .data.role.lower();
    }

    function roleIsStaff() {
      return adminRole() in ['owner', 'admin', 'staff'];
    }

    // the single gate every privileged write goes through
    function isStaff() {
      return hasAdminClaim() || (hasAdminDoc() && roleIsStaff());
    }

    // owners manage people; staff manage the shop
    function isOwner() {
      return hasAdminClaim() || (hasAdminDoc() && adminRole() == 'owner');
    }

    // the shop content lives in one known document id
    function isSiteDoc() {
      return request.resource.data.id == 'site';
    }

    // shape checks: keep junk (and hostile payloads) out of the database
    function hasOrderShape() {
      let o = request.resource.data;
      return o.status == 'pending' &&
             o.items is list &&
             o.items.size() > 0 && o.items.size() <= 40 &&
             o.customer is map &&
             o.customer.name is string && o.customer.name.size() > 1 && o.customer.name.size() <= 120 &&
             o.customer.phone is string && o.customer.phone.size() <= 40 &&
             o.customer.email is string && o.customer.email.size() <= 160 &&
             o.subtotal is number && o.tax is number && o.total is number &&
             o.subtotal >= 0 && o.tax >= 0 && o.total >= 0 &&
             (!('notes' in o) || (o.notes is string && o.notes.size() <= 2000)) &&
             (!('booking' in o) ||
               (o.booking is map &&
                o.booking.service is string && o.booking.service.size() > 0 && o.booking.service.size() <= 80 &&
                o.booking.year is string && o.booking.year.size() == 4 &&
                o.booking.vehicle is string && o.booking.vehicle.size() <= 100 &&
                o.booking.date is string && o.booking.date.size() <= 10 &&
                o.booking.time is string && o.booking.time.size() <= 40 &&
                o.booking.notes is string && o.booking.notes.size() <= 1500)) &&
             (!('code' in o) || (o.code is string && o.code.size() <= 24));
    }

    function hasLineShape() {
      let items = request.resource.data.items;
      let item = items[0];
      return items.size() == 1 &&
        item.id is string && item.id.size() > 0 && item.id.size() <= 120 &&
        item.quantity is int && item.quantity > 0 && item.quantity <= 99 &&
        item.price is number && item.price >= 0 && item.price <= 500000;
    }

    function hasMessageShape() {
      let m = request.resource.data;
      return m.name is string && m.name.size() > 1 && m.name.size() <= 120 &&
             m.email is string && m.email.size() <= 160 &&
             m.message is string && m.message.size() > 4 && m.message.size() <= 4000 &&
             m.status == 'new' &&
             (!('phone' in m) || (m.phone is string && m.phone.size() <= 40)) &&
             (!('subject' in m) || (m.subject is string && m.subject.size() <= 40));
    }

    /* ================= products ================= */

    match /products/{productId} {
      // the shop is a public catalogue: browseable by anyone
      allow read: if true;

      // Exact inventory counts belong only in the private inventory collection.
      allow create, update: if isStaff() &&
        !('stock' in request.resource.data) &&
        request.resource.data.nameEs is string &&
        request.resource.data.nameEs.size() > 0 &&
        request.resource.data.nameEs.size() <= 120 &&
        request.resource.data.categoryEs is string &&
        request.resource.data.categoryEs.size() > 0 &&
        request.resource.data.categoryEs.size() <= 60 &&
        request.resource.data.desc is string &&
        request.resource.data.desc.size() <= 500 &&
        request.resource.data.descEs is string &&
        request.resource.data.descEs.size() <= 500 &&
        (!('imageData' in request.resource.data) ||
          (request.resource.data.imageData is string && request.resource.data.imageData.size() <= 850000));
      allow delete: if isStaff();
    }

    /* ================= product categories ================= */

    match /categories/{categoryId} {
      allow read: if true;
      allow create, update: if isStaff() &&
        request.resource.data.name is string &&
        request.resource.data.name.size() > 0 &&
        request.resource.data.name.size() <= 60 &&
        request.resource.data.nameEs is string &&
        request.resource.data.nameEs.size() > 0 &&
        request.resource.data.nameEs.size() <= 60;
      allow delete: if isStaff();
    }

    /* ================= private inventory ================= */

    match /inventory/{productId} {
      // Exact wheel quantities are staff-only; the public product record has no stock field.
      allow read: if isStaff();
      allow create, update: if isStaff() &&
        request.resource.data.stock is int &&
        request.resource.data.stock >= 0 &&
        request.resource.data.stock <= 1000000;
      allow delete: if isStaff();
    }

    /* ================= content (CMS) ================= */

    match /content/{contentId} {
      allow read: if contentId == 'site';
      allow write: if isStaff() && isSiteDoc();
    }

    /* ================= orders ================= */

    match /orders/{orderId} {
      // never listable or readable by the public — an order doc holds a name,
      // a phone number and a home address
      allow read: if isStaff();

      // a visitor may file exactly one thing: a new, pending quote.
      // They cannot read back the collection, cannot touch an existing order,
      // and cannot set any status other than 'pending'.
      allow create: if hasOrderShape() && hasLineShape();

      // Staff may confirm/cancel bookings or mark one sold; the dashboard uses
      // transactions so status changes and stock reservations/releases happen together.
      // Customers can never change a booking or stock.
      allow update: if isStaff();

      allow delete: if isOwner();
    }

    /* ================= contact messages ================= */

    match /messages/{messageId} {
      allow read: if isStaff();
      allow create: if hasMessageShape();
      allow update, delete: if isStaff();
    }

    /* ================= customers ================= */

    match /customers/{customerId} {
      allow read, write: if isStaff();
    }

    /* ================= admin roles ================= */

    match /admins/{uid} {
      // a signed-in user may read their own role document; that is the call
      // assets/js/firebase-config.js makes to resolve a role without custom claims
      allow get: if signedIn() && (request.auth.uid == uid || isOwner());
      allow list: if isOwner();

      // only owners hand out access, and nobody can grant themselves
      allow create, update: if isOwner() &&
        request.resource.data.role in ['owner', 'admin', 'staff'] &&
        request.resource.data.email is string;
      allow delete: if isOwner();
    }

    /* ================= everything else ================= */

    // deny by default: any collection added later is locked until you write a rule
    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

### Why a customer's total is not blindly trusted

`hasOrderShape()` forces `status == 'pending'`, bounds every string, and rejects a
payload without items — but Firestore rules cannot re-run your price book for every
line (rules have a 10-document-read budget per evaluation, and no loop). The order the
site files is therefore a **quote**, which is exactly what the checkout page tells the
customer: *"Final shipping and tax are confirmed by our team before payment."* A
customer editing prices in devtools gains nothing: no payment is taken, staff see the
number in the dashboard beside the phone call, and only staff can move it out of
`pending`.

If you want the strictest version anyway — a rule that refuses an order whose first
line does not match the live catalogue price — replace `hasLineShape()` with:

```
    function firstLineMatchesCatalog() {
      let item = request.resource.data.items[0];
      return exists(/databases/$(database)/documents/products/$(item.id)) &&
             get(/databases/$(database)/documents/products/$(item.id)).data.price == item.price;
    }
```

and add it to the `allow create`. The honest limits: it only checks the **first** line
(no iteration over document reads), it adds one read to every checkout, and it breaks
legitimate carts where a shopper added an item before an admin repriced it. The
production-grade answer is a ~20-line Cloud Function that re-reads the products and
rewrites `subtotal` / `tax` / `total` on create; that is deliberately **not** wired up
here because the brief keeps payments optional and out of scope.

---

## 4. Legacy Storage rules (not used)

The current site does not use Firebase Storage. Product photos are resized and
compressed in the browser, then saved as `imageData` in each public Firestore product
document. Do not configure or publish Storage rules for this feature. The rules below
are retained only for reference by older deployments.

Publish from *Firebase Console → Storage → Rules*.

```
rules_version = '2';

service firebase.storage {
  match /b/{bucket}/o {

    function signedIn() {
      return request.auth != null;
    }

    // storage rules reach Firestore with firestore.get() / firestore.exists()
    function hasAdminClaim() {
      return signedIn() &&
        (request.auth.token.admin == true || request.auth.token.role == 'admin');
    }

    function hasStaffRoleDoc() {
      return signedIn() &&
        firestore.exists(/databases/(default)/documents/admins/$(request.auth.uid)) &&
        firestore.get(/databases/(default)/documents/admins/$(request.auth.uid)).data.role in
          ['owner', 'admin', 'staff'];
    }

    function isStaff() {
      return hasAdminClaim() || hasStaffRoleDoc();
    }

    function looksLikeImage() {
      return request.resource.size < 8 * 1024 * 1024 &&
             request.resource.contentType.matches('image/(jpeg|png|webp|avif|gif)') &&
             (request.resource.name.matches('[a-zA-Z0-9._-]{1,120}') ||
              request.filePath.matches('products/[a-zA-Z0-9._-]{1,140}'));
    }

    match /products/{imageName} {
      // product photos are public: they are the storefront
      allow read: if true;
      // uploads come only from the admin product editor
      allow write: if isStaff() && looksLikeImage();
    }

    // nothing else may be uploaded or read
    match /{allPaths=**} {
      allow read, write: if false;
    }
  }
}
```

Notes that matter in practice:

* `firestore.get()` inside **storage** rules is correct there (in *Firestore* rules the
  same call is plain `get()` — mixing the two up makes the console refuse to publish).
* If you set the custom claim `{"admin": true}` on every staff login, `hasAdminClaim()`
  answers alone and the Firestore lookup never runs.
* The current frontend does not upload files to Storage.

---

## 5. Optional: lock down the anonymous side further

By default Firebase enables **Email/Password** sign-in for anyone, so a stranger can
create a free account on your project. That account is harmless under these rules (no
admin claim, no `admins` doc — every privileged write is refused), but it still puts
junk rows in *Authentication → Users*. Two tidy-ups:

1. Firebase Console → *Authentication* → *Settings* → **Authorized domains**: keep only
   `localhost`, `127.0.0.1`, your `*.vercel.app` domain and your custom domain.
2. Only staff should ever sign in. Customers never need an account on this site — the
   checkout and contact forms are public and write only `orders` / `messages`.

---

## 6. Getting the first admin in (no claims, chicken-and-egg)

The rules let an owner write `admins/*`, but the very first staff document has no owner
yet. Pick one:

**A. Custom claim (fastest, recommended).**
Firebase Console → *Authentication* → *Users* → open your own account → ⋮ →
**Edit claims** → `{ "admin": true }` → Save → sign out and back in (the claim rides on
a fresh ID token). Now `isStaff()` is true in the rules and the private dashboard can
read booking and inventory data.

**B. One manual document.**
Firestore → *Start collection* → collection id `admins` → document ID = the uid from
*Authentication → Users* (looks like `Xk2f…`) → fields:

| field | type | value |
| --- | --- | --- |
| `email` | string | their exact sign-in email |
| `role` | string | `owner` |
| `createdAt` | timestamp | now |

Add a matching role document for every additional staff UID that needs dashboard access.

---

## 7. Deploying the rules from the command line

```
# one-time
npm install -g firebase-tools
firebase login
firebase init projects/firebase-bigboys        # choose Firestore + Storage only
```

`firebase init` creates that `projects/firebase-bigboys` folder (the name is yours to
pick) with a `firebase.json` inside it; the site itself needs no Firebase files, so
many owners keep this folder outside the website repo.

`projects/firebase-bigboys/firebase.json`:

```json
{
  "firestore": { "rules": "firestore.rules", "indexes": "firestore.indexes.json" },
  "storage": { "rules": "storage.rules" }
}
```

Save the two rule bodies above as `projects/firebase-bigboys/firestore.rules` and
`projects/firebase-bigboys/storage.rules`, then:

```
firebase deploy --only firestore:rules,storage:rules
```

This never touches your data and never redeploys the website. It also means the rules
live in git next to the site instead of only in the console.

---

## 8. Verify it, don't assume it

The console's rules editor does not prove what is enforced, so test by effect:

1. **Signed out**, browse products and submit a booking request. The public product
   documents must not contain a `stock` field.
2. **Signed out**, try to read the `orders` and `inventory` collections through the
   Firestore REST API — both must return `PERMISSION DENIED`:
   ```js
   fetch('https://firestore.googleapis.com/v1/projects/YOUR-PROJECT-id/databases/(default)/documents/orders?key=WEB-API-KEY')
     .then(r => r.json()).then(console.log)
   ```
3. Sign in as staff and confirm adding, editing, and deleting products changes the
   public catalogue while exact stock remains visible only in the dashboard.
4. Create a product booking and verify pending status does not change stock. Confirm it
   and verify stock decreases by the requested quantity. Cancel the confirmed booking
   and verify that quantity is returned. Confirm another booking and mark it sold;
   verify stock is not deducted twice.
5. Submit another booking from a separate browser; verify it appears in the signed-in
   dashboard with customer details. Sign out and confirm those details disappear.

---

## 9. Credentials checklist (the part that gets people breached)

* `FIREBASE_CONFIG` in `index.html` is **public by design** — the web
  API key identifies your project, it does not authorize anything. The rules do that.
* Never put `serviceAccountKey.json`, a Google private key, an Admin SDK credential, a
  Firebase Auth *admin* token, or a `.env` file in this repository or in `assets/`.
  EmailJS public identifiers are intended for browser use, but SMTP passwords and other
  private credentials must never be placed in this repository.
* Turn on *App Check* (reCAPTCHA Enterprise is free for modest traffic) if you later find
  bots spamming the contact form — it stops requests from outside your domain.
* Firestore → *Usage* → turn **test features off** once you are live; that removes the
  "anyone can write for 30 days" window a new database sometimes ships with.
