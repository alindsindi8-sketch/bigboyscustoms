# BIGBOYS CUSTOMS — Custom Wheels & Tires

**"We are your one stop shop."** · 3100 Mansfield Hwy, Forest Hill, TX 76119 · **817-535-2284** · OPEN 7 DAYS A WEEK
Email **Bigboyscustoms@live.com** · [Instagram](https://www.instagram.com/bigboyscustoms/) ·
[TikTok](https://www.tiktok.com/@bigboyscustoms) · [YouTube](https://www.youtube.com/@bigboyscustoms)

This workspace currently runs as one `index.html` file with hash-based Home, Shop,
Services, Why Us, Contact, and Staff Dashboard views. It includes separate customer
booking and cart-order forms, plus Firebase-backed product and stock management.
In the staff dashboard, product orders and service bookings are shown on separate
Product orders and Service bookings screens.

**Firebase is already configured in this copy.** The public web config values in the
inline `FIREBASE_CONFIG` object in `index.html` are present. Publish the rules in
[`firebase-rules.md`](firebase-rules.md) before relying on online requests or live
inventory. Never treat preview inventory as live stock.

The site uses **HTML, CSS, vanilla JavaScript, and Firebase** and can be served as static
files; the local server is for preview only.

---

## Contents

1. [Running it right now (nothing to install)](#1-vs-code--run-it-in-30-seconds)
2. [Create the Firebase project](#2-create-the-firebase-project)
3. [Enable Firestore](#3-enable-cloud-firestore)
4. [Enable Authentication](#4-enable-authentication)
5. [Product photos (no Storage needed)](#5-product-photos-no-storage-needed)
6. [Give yourself admin access — securely](#6-configure-admin-access-securely)
7. [Deploy to Vercel (free)](#7-deploy-to-vercel-free)
8. [Connect your custom domain](#8-connect-a-custom-domain)
* [Booking and order email notifications](#booking-and-order-email-notifications) · [Preview mode and staff dashboard](#preview-mode-and-staff-dashboard) · [Payments and customer data](#payments-and-customer-data) · [Current files](#current-files) · [Troubleshooting](#troubleshooting) · [Security rules](#security-rules)

---

## 1. VS Code — run it in 30 seconds

1. Install [VS Code](https://code.visualstudio.com/).
2. In VS Code: **File → Open Folder…** and pick the `big boys` folder.
3. Install the **Live Server** extension (Extensions sidebar → search `Live Server`,
   author *Ritwick Dey* → Install).
4. Right-click `index.html` in the Explorer → **Open with Live Server**.

You get `http://127.0.0.1:5500/index.html`. Staff sign-in is at `/#admin`.

Two things to know while developing:

* **Use a server, not `file://`.** Firebase SDK modules are loaded with
  `import()`, which browsers refuse to run from a `file://` page. Live Server,
  `npx serve`, or `python -m http.server` all work; opening the HTML by double-clicking
  shows the site but not the live-data features.
* **If Firebase is unavailable:** sample cards display a Demo badge; the booking and
  order forms explain that requests are not stored until the connection is restored.

## 2. Create the Firebase project

1. Go to <https://console.firebase.google.com/> → **Add project**.
2. Name it `bigboys-customs`. Google Analytics is optional (skip it).
3. When the project opens, press the **`</>` (Web)** button and register an app named
   `bigboys-web`. **Do not** tick "Firebase Hosting" — Vercel serves the site.
4. Firebase now shows a `firebaseConfig` object with ~7 string values. Copy it and paste
   it into the `FIREBASE_CONFIG` object in `index.html`. Those values are public by design (they identify your
   project). The thing you must never paste here is a **service account key** — see
   [Security rules](#security-rules).
5. In `index.html`, find the inline `FIREBASE_CONFIG` object near the bottom and paste
   the Firebase Web app values into its fields. `storageBucket` is optional and may stay
   blank because this site does not use Firebase Storage:

   ```js
   const FIREBASE_CONFIG = {
     apiKey: '...',
     authDomain: '...',
     projectId: '...',
     storageBucket: '...',
     messagingSenderId: '...',
     appId: '...'
   };
   ```

   The Firebase Web config is public project-identification data. Never put a password,
   service-account key, private token, or admin credential in the page.

## 3. Enable Cloud Firestore

1. Console left sidebar → **Build → Firestore Database → Create database**.
2. Region: **`us-central`** or the closest to Texas (`us-east1` is a good pick).
3. Start in **production mode** (locked). The rules in
   [`firebase-rules.md`](firebase-rules.md) allow public catalogue reads and booking
   and product-order submissions, while protecting customer details and exact stock counts.

Create and update these collections through the site:

| collection | what lands in it |
| --- | --- |
| `products` | public product details and general availability (in stock / low stock / out of stock), without exact stock counts |
| `categories` | shop categories such as Wheels & Tires, Lift Kits, and Drop Kits |
| `inventory` | exact stock counts, readable and writable only by authorized staff |
| `orders` | booking and cart-order requests, starting at `pending`; private to staff for reading |
| `admins` | `{your-uid}` with `role: "owner"` |

When staff confirm an order, inventory is reduced in the same Firestore transaction
that confirms the order; cancelling a confirmed order returns the quantity. The
dashboard shows exact counts, while the public shop shows only **In stock**, **Low
stock** (three or fewer), or **Out of stock**. Existing products synchronize their
availability the first time authorized staff sign in after publishing the updated site
and rules.

## 4. Enable Authentication

1. Console → **Build → Authentication → Get started**.
2. **Sign-in method** tab → **Email/Password** → enable → Save.
   No other provider is used by this site.
3. Create the admin account in **Users** → **Add user** using
   `bigboys@admin.com`. Passwords are never stored in the website code.
4. **Settings** tab → **Authorized domains**: add your `*.vercel.app` domain and your
   custom domain when you have them. `localhost` is there already.
5. Leave "Email link / Google / Anonymous" off — this dashboard is staff-only and
   customers never create accounts.

## Booking and order email notifications

Booking and product-order emails use EmailJS from the browser, so Firebase Cloud
Functions and the Firebase Blaze billing plan are not required. Both notifications go to
the same fixed shop address, but use separate templates and separate forms. A booking
uses the appointment form and booking template; a cart order uses the header cart, a
separate contact/fulfillment form, and the order template. Each request is saved in
Firestore first. Product orders appear in the staff dashboard and trigger an owner email.
When staff confirm a product order, the site sends the customer an order-confirmation
email with the order items, fulfillment details, and stored total. Email failures do not
undo a saved or confirmed order; the dashboard reports a failed confirmation email.

1. Create a free account at [EmailJS](https://www.emailjs.com/).
2. In EmailJS, add an email service and connect the sending mailbox. If you use Gmail,
   authorize it through EmailJS; never put your Gmail password in this project.
3. Create a booking email template. Set the **To email** field directly to
   `alindisindi8@gmail.com` (do not use a customer-provided value for the recipient).
   Include these template variables in the subject/body:
   `{{customer_name}}`, `{{customer_email}}`, `{{customer_phone}}`, `{{service}}`,
   `{{vehicle}}`, `{{date}}`, `{{time}}`, `{{notes}}`, and `{{booking_link}}`.
   A suggested message is: “You have a new booking on your website. Please review it:
   {{booking_link}}”.
4. Create a separate product-order email template using the same fixed **To email**
   recipient. Include `{{customer_name}}`, `{{customer_email}}`, `{{customer_phone}}`,
   `{{order_id}}`, `{{items}}`, `{{items_html}}`, `{{subtotal}}`, `{{tax}}`, `{{total}}`,
   `{{fulfillment_method}}`, `{{delivery_location}}`, `{{vehicle}}`, `{{notes}}`, and
   `{{order_link}}`. Give it a distinct subject and design so order alerts are easy to
   distinguish from booking alerts.
5. Create a third template for **customer order confirmations**. Set its **To email**
   field to `{{to_email}}`. Include `{{customer_name}}`, `{{customer_email}}`,
   `{{customer_phone}}`, `{{order_id}}`, `{{items}}`, `{{items_html}}`, `{{subtotal}}`, `{{tax}}`,
   `{{total}}`, `{{fulfillment_method}}`, `{{delivery_location}}`, `{{vehicle}}`,
   `{{notes}}`, and `{{order_status}}`. The customer email is sent only after staff
   confirm a pending product order in the dashboard. Add `{{items_html}}` in the
   template's HTML body where the formatted product list and available product photos
   should appear. Email apps may block embedded image data; use publicly hosted image
   URLs if a recipient's email app does not display those photos.
6. In EmailJS account settings, copy the **Public Key**. Also copy the service ID and
   all three template IDs from their respective pages.
7. In `index.html`, set `serviceId`, `templateId` (booking), `orderTemplateId` (owner
   product-order alert), `orderConfirmationTemplateId` (customer confirmation), and
   `publicKey` in `EMAILJS_CONFIG`. These are browser-side public identifiers, not
   passwords. Do not put SMTP passwords, private keys, or other secrets in the page.
8. Publish the updated website. Submit a test cart order and check the owner inbox,
   then sign in to the staff dashboard and confirm that order. Check the customer's
   inbox for the confirmation and verify the items and total. Check Spam and EmailJS
   Email History if either message does not arrive.

Because EmailJS sends from the browser, set its allowed origins/domain restrictions in
the EmailJS dashboard to your website domain, and keep the template recipient fixed to
the owner’s address. Check the EmailJS dashboard for current free-tier limits and
delivery status.

## 5. Product photos (no Storage needed)

The admin can upload a product photo without enabling Firebase Storage. The browser
resizes it and stores it in the Firestore product document at no more than 600 KB.
PNG files stay PNG (including existing transparency); a plain-color background
connected to the image edges is removed locally when detected. Complex backgrounds
may still need to be removed before upload. Other supported formats are stored as
JPEG. Keep photos reasonably sized; Firestore documents have a 1 MiB maximum. The
built-in wheel artwork remains the fallback when a product has no photo.

## 6. Configure admin access securely

Firestore rules protect the data even if someone bypasses the website UI. Two approved
ways to grant staff access are supported:

**A. Custom claim — recommended.** Console → **Authentication → Users** → open yourself
→ ⋮ → **Edit claims** → paste:

```json
{ "admin": true }
```

Save, then **sign out and sign back in** — the claim is baked into the ID token at
sign-in, so an already-open tab keeps its old token.

**B. Role document** — Firestore → `admins` collection → **Add document** with Document
ID equal to your UID (Authentication → Users → *UID*) and fields `email` (string) and
`role` (string `owner`). Staff accounts must also be explicitly granted a staff role in
this collection or via a custom claim. The website itself cannot grant admin privileges.

Now publish [`firebase-rules.md`](firebase-rules.md) §3 into Firestore → Rules →
**Publish**. Finally run the verification checks in its §8 — the console's editor will
happily save rules that are wrong, so the only real proof is behaviour.

## 7. Deploy to Vercel free

Option 1 — dashboard (no local tooling):

1. <https://vercel.com/new> on its own is fine: choose **Import a repository**, or
2. drag the `big boys` folder up as a project after pushing it to GitHub — the cleanest
   long-term path is GitHub → Vercel, so every future edit is a `git push`.

Option 2 — from your machine:

```
npm i -g vercel
cd "C:/Users/judi/Desktop/big boys"
vercel
```

Answer the prompts: link to your account, project name `bigboys-customs`, framework
preset **Other**, everything else **Enter** (no build command, no install, output
directory `.`). Run `vercel` again later for a preview, `vercel --prod` to publish.

No build step exists. Keep `bg.mp4` and `BigBoysCustoms_Intro.mp4` next to `index.html`.
The intro video plays on the home page and can be skipped. [`vercel.json`](vercel.json)
only sets cache headers and clean URLs; deleting the file is harmless.

Then finish the loop: Firebase → Authentication → Settings → **Authorized domains** →
add `your-app.vercel.app`.

## 8. Connect a custom domain

1. Vercel dashboard → your project → **Settings → Domains → Add**.
2. Type `bigboyscustoms.com` (it offers `www.` too).
3. Vercel shows two/three DNS records. In your registrar (GoDaddy, Namecheap, Cloudflare)
   add them: `A  76.76.21.21` for the apex, `CNAME  cname.vercel-dns.com` for `www`.
   A Cloudflare proxy must be set to **DNS only** — an orange-cloud proxy breaks the
   Vercel certificate.
4. Wait for **Valid configuration** (often 5–20 minutes, occasionally up to a day).
   Vercel issues the SSL certificate automatically; HTTP redirects to HTTPS by itself.
5. Set the domain you want as canonical (Vercel offers a redirect toggle).
6. Back in Firebase: add the new domain to **Authorized domains**.

---

## Preview mode and staff dashboard

Before Firebase is configured, the shop displays four clearly marked sample products.
The booking form does not store or send customer information, and staff sign-in is
disabled. This is intentional: browser-only storage would not be private or shared with
the admin.

After Firebase is configured and the Firestore rules in `firebase-rules.md` are
published:

* Customers can submit booking requests with contact details, vehicle information,
  desired product or service, preferred date/time, and notes.
* Staff open the **Staff sign in** link in the footer (or visit `/#admin`) and sign in
  with Firebase Email/Password. Only an authorized staff account can read booking data.
* Staff can add categories and products in English and Spanish, including Wheels & Tires,
   Lift Kits, and Drop Kits. Visitors can switch languages with the globe button; product
   photos are stored in Firestore without Firebase Storage. Shop cards show live stock and
   can be filtered by category.
* A pending booking does not reduce inventory. When staff confirm a product booking,
  its requested quantity is deducted from available stock in the same Firestore
  transaction as the status change. Cancelling or deleting a confirmed booking returns
  its reserved quantity to stock. Marking a pending booking sold deducts stock; marking
  an already confirmed booking sold does not deduct it a second time.
* The dashboard is responsive on mobile; its tables can scroll horizontally when
  necessary.

The first owner must be authorized in Firebase before signing in: either set the
`admin: true` custom claim on the Firebase Auth user, or create the matching
`admins/{uid}` document with `role: "owner"`. Publish the rules before accepting real
bookings; the rules keep booking reads and inventory writes staff-only.

## Payments and customer data

The booking form is a request only; it does not charge a customer or reserve stock.
Staff confirm availability and price directly. Customer contact and vehicle details
are written to the private `orders` collection only after Firebase is configured.
Do not use the site for real bookings until the rules are published and tested.

## Current files

```
big boys/
├── index.html         storefront, page views, booking form, staff dashboard
├── bg.mp4             home hero background video (keep beside index.html)
├── BigBoysCustoms_Intro.mp4 full-screen home-page intro (keep beside index.html)
├── firebase-rules.md  Firestore security rules to publish
├── vercel.json        static hosting headers
├── style.css          legacy stylesheet (not loaded by the current page)
├── animations.css     legacy stylesheet (not loaded by the current page)
└── README.md          setup and usage
```

The wheel illustrations, page styles, animations, Firebase SDK integration, and page
logic are embedded in `index.html`. Keep both video files beside it: `bg.mp4` plays
muted and loops behind the home hero, while `BigBoysCustoms_Intro.mp4` plays as the
skippable full-screen home-page intro.

## Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| Booking form says Firebase is not connected | Add the Web config values to the inline `FIREBASE_CONFIG` object in `index.html`; `storageBucket` may stay blank. |
| Firebase cannot start after paste | Check for a missing quote or comma in the config object and compare its values with the Firebase console. |
| Staff sign-in is denied | Grant the Firebase Auth UID an admin claim or a matching `admins/{uid}` role document, and publish the Firestore rules. |
| Shop has no products after Firebase is connected | Add products in `/#admin` after signing in; preview inventory is not copied into Firestore. |
| Booking submit is denied | Confirm Firestore rules §3 are published and the booking form includes valid customer and vehicle information. |
| Mobile menu or animation janky on an old phone | The site honours `prefers-reduced-motion`; enable that OS setting to confirm the layout still reads correctly without motion. |

## Security rules

Read [`firebase-rules.md`](firebase-rules.md) before you tell anyone the site is live.
It contains the publishable Firestore rules, the reasoning behind each one,
the first-admin bootstrap, how to deploy rules from the CLI, checks that prove
enforcement by effect rather than by reading the console, and the credentials checklist.
EmailJS uses public browser identifiers; never put SMTP passwords or private credentials
in this folder or the website code.
