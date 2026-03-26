# Po & Jeanine Wedding Website — Setup Guide

## Quick Start (No Backend)

The website works immediately out of the box! Open `wedding-website.html` in a browser and RSVPs will be saved to your browser's local storage. This is great for previewing and testing.

For a production setup with persistent data and guest messaging, follow the steps below.

---

## 1. Free Hosting with GitHub Pages

1. Create a free GitHub account at github.com
2. Create a new repository (e.g., `po-and-jeanine-wedding`)
3. Upload `wedding-website.html` and rename it to `index.html`
4. Go to **Settings → Pages → Source** and select "main" branch
5. Your site will be live at `https://yourusername.github.io/po-and-jeanine-wedding`

**Custom domain (optional):** You can buy a domain (e.g., `poandjeaning2026.com`) from Namecheap or Google Domains (~$12/year) and point it to your GitHub Pages site.

---

## 2. Firebase Backend (Free Tier — RSVP Storage)

Firebase's free Spark plan is more than enough for a wedding website.

### Setup Steps

1. Go to [console.firebase.google.com](https://console.firebase.google.com)
2. Click **Add Project** → name it (e.g., `po-jeanine-wedding`)
3. Skip Google Analytics (not needed)
4. Once created, click **Web** (</>) to add a web app
5. Copy the config object and paste it into the `FIREBASE_CONFIG` section of your HTML file:

```javascript
const FIREBASE_CONFIG = {
  apiKey: "AIzaSy...",
  authDomain: "po-jeanine-wedding.firebaseapp.com",
  projectId: "po-jeanine-wedding",
  storageBucket: "po-jeanine-wedding.appspot.com",
  messagingSenderId: "123456789",
  appId: "1:123456789:web:abc123"
};
```

6. In the Firebase console, go to **Firestore Database** → **Create Database** → Start in **production mode**
7. Set up security rules (paste into Rules tab):

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /rsvps/{doc} {
      allow create: if true;       // Anyone can submit RSVP
      allow read: if true;         // Admin panel can read
    }
    match /announcements/{doc} {
      allow read: if true;
      allow create: if true;
    }
  }
}
```

---

## 3. Guest Messaging (Email + SMS)

Messaging requires a small backend. The easiest approach is Firebase Cloud Functions.

### Email via SendGrid (Free: 100 emails/day)

1. Sign up at [sendgrid.com](https://sendgrid.com)
2. Create an API key under **Settings → API Keys**
3. Verify a sender email address

### SMS via Twilio (Free trial: ~$15 credit)

1. Sign up at [twilio.com](https://www.twilio.com)
2. Get your Account SID, Auth Token, and a phone number
3. Verify recipient numbers during trial period

### Deploy Cloud Function

1. Install Firebase CLI: `npm install -g firebase-tools`
2. Run `firebase init functions` in your project folder
3. Create `functions/index.js`:

```javascript
const functions = require('firebase-functions');
const admin = require('firebase-admin');
const sgMail = require('@sendgrid/mail');
const twilio = require('twilio');

admin.initializeApp();

// Configure these with: firebase functions:config:set
// sendgrid.key="SG.xxx" twilio.sid="AC..." twilio.token="..." twilio.phone="+1..."

exports.sendMessage = functions.https.onRequest(async (req, res) => {
  res.set('Access-Control-Allow-Origin', '*');
  res.set('Access-Control-Allow-Methods', 'POST');
  res.set('Access-Control-Allow-Headers', 'Content-Type');
  if (req.method === 'OPTIONS') { res.status(200).send(''); return; }

  const { recipients, type, subject, body } = req.body;
  const config = functions.config();

  // Get guest list from Firestore
  const snapshot = await admin.firestore().collection('rsvps').get();
  let guests = snapshot.docs.map(doc => doc.data());

  // Filter by recipient type
  if (recipients === 'attending') guests = guests.filter(g => g.attending === 'yes');
  if (recipients === 'not-responded') guests = []; // Would need invite list comparison

  // Send emails
  if ((type === 'email' || type === 'both') && config.sendgrid?.key) {
    sgMail.setApiKey(config.sendgrid.key);
    const emails = guests.filter(g => g.email).map(g => ({
      to: g.email,
      from: 'your-verified@email.com', // Change this
      subject: subject || 'Update from Po & Jeanine',
      text: body,
      html: `<div style="font-family:sans-serif;max-width:600px;margin:0 auto;">
        <h2 style="color:#C4A35A;">Po & Jeanine</h2>
        <p>${body.replace(/\n/g, '<br>')}</p>
        <hr style="border:1px solid #eee;margin:2rem 0;">
        <p style="color:#999;font-size:12px;">August 23, 2026 · The Great American Beer Hall · Malden, MA</p>
      </div>`
    }));
    await Promise.all(emails.map(e => sgMail.send(e)));
  }

  // Send SMS
  if ((type === 'sms' || type === 'both') && config.twilio?.sid) {
    const client = twilio(config.twilio.sid, config.twilio.token);
    const smsGuests = guests.filter(g => g.phone);
    await Promise.all(smsGuests.map(g =>
      client.messages.create({
        body: body,
        from: config.twilio.phone,
        to: g.phone
      })
    ));
  }

  res.json({ success: true, sent: guests.length });
});
```

4. Install dependencies: `cd functions && npm install @sendgrid/mail twilio`
5. Set config: `firebase functions:config:set sendgrid.key="SG.xxx" twilio.sid="AC..." twilio.token="..." twilio.phone="+15551234567"`
6. Deploy: `firebase deploy --only functions`
7. Copy the function URL into your HTML file's `MESSAGING_API` variable

---

## 4. Admin Panel Access

Press **Ctrl + Shift + A** on the website to open the admin panel. The default password is `poAndJeanine2026` — change it in the HTML file's `ADMIN_PASSWORD` variable.

From the admin panel you can:
- View all RSVPs with attending/declining counts
- Send email and SMS messages to guests
- Post new announcements to the website

---

## 5. Customization Quick Reference

| What to Change | Where in the HTML |
|---|---|
| Couple names | Search for "Po" and "Jeanine" |
| Wedding date | Search for "August 23, 2026" and the countdown date |
| Venue name | Search for "Great American Beer Hall" |
| Color palette | CSS `:root` variables at the top |
| RSVP deadline | The paragraph above the RSVP form |
| Our Story text | The `story-text` paragraph |
| Admin password | `ADMIN_PASSWORD` constant |
| Firebase config | `FIREBASE_CONFIG` object |

---

## Estimated Costs

| Service | Cost |
|---|---|
| GitHub Pages hosting | Free |
| Custom domain | ~$12/year (optional) |
| Firebase Spark (Firestore) | Free (up to 1GB storage) |
| SendGrid (email) | Free (100 emails/day) |
| Twilio (SMS) | ~$1/month + $0.0079/text |
| **Total** | **$0 – $15/month** |
