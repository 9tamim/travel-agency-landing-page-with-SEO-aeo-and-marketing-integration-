# Meltemi Travel: Make.com automation setup

The page sends every lead to one Make **Custom webhook**. Make then routes it by `type`.

## 1. Connect the page
1. In Make, create a scenario and add **Webhooks > Custom webhook**. Copy its URL.
2. In `index.html`, set `MAKE.url` to that URL and `MAKE.token` to a random string.
3. Host the page on your own domain. The preview here blocks outside requests.
4. Submit a test booking so Make learns the data structure.

The page posts a form field named `payload` (a JSON string), so there are no CORS problems. Add **JSON > Parse JSON** right after the webhook with `{{1.payload}}`.

### Payload
```json
{"type":"booking_request | abandoned_booking | newsletter",
 "id":"unique event id","ts":"ISO time","page":"url","ref":"referrer",
 "utm":{"utm_source":"","utm_campaign":"","gclid":""},"token":"your token",
 "data":{"ref":"MT-AB12CD","trip":"","trip_id":"","tier":"","travellers":2,
  "date":"YYYY-MM-DD","airport":"","addons":"visa,insurance","total":1240,
  "name":"","email":"","phone":"","marketing":true,"consent":true}}
```

## 2. Scenario A: inbound (instant)
1. Custom webhook, then Parse JSON.
2. **Filter:** `token` equals your token, and `data.consent` is true for bookings.
3. **Router** on `type`:

**Route booking_request**
- Code module (script below) to build the reference, deposit, balance date and email subject.
- Data store: add record (key = email, status `booked`). This also marks any earlier abandoned record as converted.
- Google Sheets: add row to a Bookings sheet.
- Stripe: create Checkout Session for the deposit; keep the payment URL.
- Gmail or SMTP: send confirmation with the deposit link, reference and itinerary.
- WhatsApp Business Cloud: send the confirmation template to the phone number.
- Salesforce: upsert the Lead and create a follow-up Task (see section 6).
- Slack or Telegram: notify the sales team. Use the `priority` field to flag big bookings.
- Google Calendar: create a hold on the departure date.
- HTTP: GA4 Measurement Protocol `purchase` or `generate_lead` event, so server-side conversions still count if the browser blocks tags. Use `gclid` for Google Ads offline conversion import.

**Route chatbot_lead** (from the on-page assistant)
- Code module, then Salesforce upsert (section 6) with `LeadSource = Chatbot` unless an ad `gclid` is present.
- Gmail or SMTP: send a quote email with the suggested trip and a link to `#book`.
- WhatsApp Business Cloud: send the welcome template.
- Slack or Telegram: alert the team with the trip, budget and phone number.
- Data store: add record (status `chat_lead`) so the 30-minute recovery scenario does not email the same person twice.

**Route abandoned_booking**
- Data store: add record (key = email, status `abandoned`, saved time). Skip it if the key already has status `booked`.

**Route newsletter**
- Brevo or Mailchimp: add subscriber with tag `website`.
- Send a welcome email with a deals link.

## 3. Scenario B: scheduled follow-ups
- **Every 30 minutes, abandoned recovery:** search Data store for `abandoned` records older than 60 minutes. Email "Your trip is waiting" with a link back to `#book`, then set status `recovered_sent`.
- **Daily 09:00, balance reminder:** find bookings where departure minus 14 days is today and status is `deposit_paid`. Email the balance payment link.
- **Daily, pre-trip:** 3 days before departure, send packing list, flight details and the guide's WhatsApp number.
- **Daily, review request:** 2 days after return, send a review request with your Google Reviews link.

## 4. Scenario C: payments
- **Stripe > Watch events** (`checkout.session.completed`): set status `deposit_paid`, email a receipt, tell the team, and send the GA4 `purchase` event.

## 5. Code module script
Add **Code > Run JavaScript**. Map the input variable `payload` to `{{1.payload}}`.

```js
const p = JSON.parse(input.payload);
const d = p.data, utm = p.utm || {};
const ref = d.ref || ('MT-' + Date.now().toString(36).toUpperCase().slice(-6));
const iso = x => x.toISOString().slice(0, 10);
const dep = d.date ? new Date(d.date) : null;
const due = dep ? new Date(dep.getTime() - 14 * 864e5) : null;
const total = Number(d.total) || 0;
const deposit = Math.round(total * 0.3);
const parts = (d.name || '').trim().split(/\s+/);
const medium = utm.utm_medium || '';
const leadSource = utm.gclid ? 'Google Ads' : utm.fbclid ? 'Meta Ads'
  : medium === 'organic' ? 'Organic Search' : medium === 'social' ? 'Social'
  : medium === 'email' ? 'Email' : d.channel === 'chatbot' ? 'Chatbot' : 'Website';
return {
  ref, type: p.type, channel: d.channel || 'booking_form',
  firstName: parts.length > 1 ? parts[0] : '',
  lastName: parts.length > 1 ? parts.slice(1).join(' ') : parts[0],
  email: (d.email || '').trim().toLowerCase(),
  phone: (d.phone || '').replace(/[^\d+]/g, ''),
  trip: d.trip, tier: d.tier || '', airport: d.airport || '',
  travellers: Number(d.travellers) || 0,
  departure: dep ? iso(dep) : '', total, deposit,
  balance: total - deposit, balanceDue: due ? iso(due) : '',
  priority: total >= 2000 ? 'High' : 'Normal',
  leadSource, utmSource: utm.utm_source || '', utmMedium: medium,
  utmCampaign: utm.utm_campaign || '', gclid: utm.gclid || '',
  marketing: !!d.marketing, consentAt: p.ts,
  description: `${d.trip} for ${d.travellers} travellers. Ref ${ref}. Source ${leadSource}.`,
  subject: `Your ${d.trip} booking ${ref}: deposit EUR ${deposit}`
};
```

Make's Code app depends on your plan. If you do not have it, use the **Tools > Set multiple variables** module with the same formulas.

## 6. Salesforce integration
Connect Make's **Salesforce** app with OAuth (use a dedicated integration user).

**Upsert logic (booking and chatbot routes)**
1. Salesforce > **Search Records** (SOQL): `SELECT Id FROM Lead WHERE Email = '{{email}}' AND IsConverted = false LIMIT 1`.
2. **Router:** found, then **Update a Record** (Lead). Not found, then **Create a Record** (Lead).
3. Salesforce > **Create a Record** (Task): Subject `Call new lead {{ref}}`, WhoId = Lead Id, due today, Priority `High` if `priority` is High.

**Lead field mapping**

| Salesforce field | Value from the Code module |
|---|---|
| FirstName, LastName | `firstName`, `lastName` |
| Email, Phone | `email`, `phone` |
| Company (required) | `Individual traveller` |
| Status | `New` (set `Working` after the deposit is paid) |
| LeadSource | `leadSource` |
| HasOptedOutOfEmail | `NOT marketing` |
| Description | `description` |
| Booking_Ref__c, Channel__c | `ref`, `channel` |
| Trip__c, Travellers__c, Departure_Date__c, Total_Value__c | `trip`, `travellers`, `departure`, `total` |
| UTM_Source__c, UTM_Medium__c, UTM_Campaign__c, GCLID__c | `utmSource`, `utmMedium`, `utmCampaign`, `gclid` |
| Consent_Timestamp__c | `consentAt` |

Create the `__c` custom fields in Salesforce first (Setup > Object Manager > Lead). Add `LeadSource` picklist values: Google Ads, Meta Ads, Organic Search, Social, Email, Chatbot, Website.

**After payment (Scenario C):** update the Lead to `Working`, and use **Make an API Call** to convert it to an Account, Contact and Opportunity (amount = total, stage `Closed Won` once the balance is paid).

## 7a. GTM and Google marketing analytics
The page pushes these to `dataLayer` (UTM values are flat keys, ready for Data Layer Variables):

| Event | Key parameters |
|---|---|
| `page_context` | `site_region`, `site_country`, `currency`, `page_type` |
| `utm_captured` | `utm_source`, `utm_medium`, `utm_campaign`, `utm_term`, `utm_content`, `utm_id`, `gclid`, `gbraid`, `wbraid`, `fbclid`, `msclkid`, `landing` |
| `view_item_list`, `select_item`, `view_item`, `add_to_wishlist` | `items[]` (GA4 ecommerce format) |
| `begin_checkout`, `checkout_step` | `value`, `currency`, `items[]` |
| `generate_lead` | `value`, `currency`, `transaction_id`, `method` (chatbot or form), `items[]`, `user_data.email`, `user_data.phone_number` |
| `chat_open`, `chat_step`, `chat_recommend`, `chat_lead` | `step`, `value` |
| `trip_finder`, `filter_tours`, `cta_click`, `scroll_depth`, `newsletter_signup` | various |

**In the GTM container**
- Create Data Layer Variables for every `utm_*` key, `gclid`, `value`, `currency`, `transaction_id`, `items` and `user_data`.
- Tags: Google tag with your GA4 `G-XXXXXXX` ID (consent: analytics_storage); GA4 Event tags for each event above (one Custom Event trigger each); Conversion Linker; Google Ads Conversion on `generate_lead` with Enhanced Conversions (user-provided data = the `user_data` variable); optional Meta Pixel.
- In GA4 Admin, register custom dimensions: `utm_source`, `utm_medium`, `utm_campaign`, `method`. Mark `generate_lead` as a key event and import it into Google Ads.
- UTM values are kept in session storage only, so the page never stores tracking IDs for longer than the visit. GA4 handles multi-session attribution once the visitor accepts cookies.
- Test with GTM Preview and GA4 DebugView before publishing the container.

## 8. Advanced sales tracking
**What the page now sends with every Make payload** (`sess`, `cid`, `score` and `signals` only after the visitor accepts cookies):

| Field | Meaning |
|---|---|
| `sess` | Anonymous session ID, also in `dataLayer` as `session_key` |
| `cid` | GA4 client ID from the `_ga` cookie |
| `score` | Lead score from behaviour: select trip +10, view details +10, save trip +5, open chat +10, start checkout +20, reach step 2 +10, promo code +5, contact click +15, read guide +5, deep scroll +5, 60 seconds engaged +5 |
| `signals` | Comma list of the behaviours behind the score |
| `data.coupon`, `data.discount` | Promo code used and amount saved |

New events: `lead_score`, `section_view`, `engaged_30s`, `contact_click`, `promo_applied`, `promo_invalid`, `exit_offer_view`. Every event carries an `event_id`, so server-side tags can de-duplicate against browser tags.

**Make additions**
- **Route `lead_magnet`:** add the subscriber to Brevo or Mailchimp with tag `exit_offer`, send the checklist and the WELCOME5 code, and create a Salesforce Lead with `LeadSource = Exit offer`.
- **Route `session_summary` and `contact_click`:** save to Data store keyed by `sess`. When a lead arrives with the same `sess`, copy `score` and `signals` onto the Salesforce Lead.
- **Salesforce:** add `Lead_Score__c` (number), `Signals__c` (text), `Session_Id__c`, `Coupon__c`, `GA_Client_Id__c`. Build a Lead assignment rule: score 40 or more goes to the senior advisor queue.
- **Closed-won back to Google:** when the Opportunity becomes Closed Won, send a GA4 Measurement Protocol `purchase` event using `GA_Client_Id__c` and the `transaction_id`. Upload the `gclid` as a Google Ads offline conversion, so ad spend is judged on real revenue.
- **Meta Conversions API (optional):** send `Lead` and `Purchase` with the same `event_id` as the browser pixel to avoid double counting.
- **Reporting:** build Salesforce reports on Lead Source, UTM campaign, score band and conversion to Closed Won. Connect GA4 and Salesforce data in Looker Studio for cost per booking by channel.

## 9. SEO, AEO and GEO files
Upload `robots.txt`, `sitemap.xml` and `llms.txt` to the site root. Replace `www.example.com` everywhere, then submit the sitemap in Google Search Console and Bing Webmaster Tools. Keep the visible answers, the FAQ schema text and `llms.txt` consistent, and update the "Last updated" date whenever prices change.

## 7. Compliance notes
- Bookings only send after the consent tickbox is ticked. The abandoned-booking message also needs that tickbox.
- Newsletter and recovery emails need marketing consent: use `data.marketing` for recovery and the explicit signup for the newsletter.
- Score, session and client IDs are only attached after cookie consent. Promo and exit-offer emails need the consent tickbox that the page already requires.
- The token is only a spam filter, because it is visible in the page source. Rate-limit in Make and ignore unknown `type` values.
- Add your privacy policy link and data retention period before launch.
