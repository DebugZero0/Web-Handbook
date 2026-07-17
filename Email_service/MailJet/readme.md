# Email Sending Setup with Mailjet (Node.js)

## 1. Objective

This module handles transactional email sending (e.g. welcome emails, email verification, notifications) for the application using **Mailjet's REST API** — not SMTP.

### Why API instead of SMTP?

| | SMTP | API (used here) |
|---|---|---|
| Protocol | Traditional mail protocol (port 25/465/587) | HTTPS REST calls |
| Speed | Slower, connection-based | Faster, stateless requests |
| Reliability | Can be blocked by ISPs/firewalls | Works over standard HTTPS |
| Features | Basic send only | Delivery tracking, templates, analytics, bounce handling |
| Setup | Requires host, port, auth credentials | Requires API Key + Secret Key |

We use Mailjet's official Node.js SDK (`node-mailjet`), which internally calls Mailjet's `v3.1` Send API over HTTPS. This is simpler to set up, avoids SMTP port-blocking issues on many hosting providers, and gives access to Mailjet's dashboard analytics (delivery, opens, bounces, etc.).

---

## 2. Mailjet Setup Requirements

Before the code will work, complete these steps on [Mailjet](https://app.mailjet.com):

### Step 1: Create a Mailjet Account
Sign up at https://app.mailjet.com if you haven't already.

### Step 2: Add & Verify Your Sending Domain
1. Go to **Account Settings → Sender domains & addresses**
2. Add your domain (e.g. `yourdomain.com`)
3. Add the DNS records Mailjet provides (SPF, DKIM, and optionally DMARC) to your domain's DNS settings
4. Wait for verification (can take a few minutes to a few hours depending on DNS propagation)"Don't be impatient — you can check the status in the Mailjet dashboard."

### Step 3: Add & Verify Sender Email Address
1. In the same **Sender domains & addresses** section, add the specific email address you'll send from (e.g. `noreply@yourdomain.com`)
2. Verify it (Mailjet sends a confirmation link/email)

> ⚠️ The sender email used in code **must exactly match** a verified sender address in your Mailjet account, or sends will fail.
> ⚠️ If the SPF record is not found the try `~all` instead of `?all` in your DNS settings for value of host name @.

### Step 4: Get API Credentials
1. Go to **Account Settings → REST API → API Key Management** (or visit https://app.mailjet.com/account/apikeys)
2. Copy the **API Key** → this is your `MJ_APIKEY_PUBLIC`
3. Reveal and copy the **Secret Key** → this is your `MJ_APIKEY_PRIVATE`

### Step 5: Add Environment Variables
Add these to your `.env` file:

```env
MJ_APIKEY_PUBLIC=your_public_key
MJ_APIKEY_PRIVATE=your_private_key
MJ_SENDER_EMAIL=noreply@yourdomain.com
MJ_SENDER_NAME=Your App Name
```

> 🔒 Never commit `.env` to version control. Add it to `.gitignore`.

### Step 6: Install Dependencies
```bash
npm install node-mailjet dotenv
```

---

## 3. Node.js Implementation (ESM)

### `mailjetClient.js`
Initializes and exports the Mailjet client instance.

```javascript
import Mailjet from 'node-mailjet';
import dotenv from 'dotenv';

dotenv.config();

const mailjet = Mailjet.apiConnect(
  process.env.MJ_APIKEY_PUBLIC,
  process.env.MJ_APIKEY_PRIVATE
);

export default mailjet;
```

### `mailController.js`
Reusable send function using positional arguments: `(to, subject, html, text, toName)`.

```javascript
import mailjet from './mailjetClient.js';

/**
 * Reusable send function
 * Usage: sendEmail(toEmail, subject, htmlContent, textContent, toName)
 */
export const sendEmail = async (to, subject, html, text, toName = '') => {
  try {
    const request = mailjet.post('send', { version: 'v3.1' }).request({
      Messages: [
        {
          From: {
            Email: process.env.MJ_SENDER_EMAIL,
            Name: process.env.MJ_SENDER_NAME,
          },
          To: [
            {
              Email: to,
              Name: toName || to,
            },
          ],
          Subject: subject,
          HTMLPart: html,
          TextPart: text,
        },
      ],
    });

    const result = await request;
    return { success: true, data: result.body };
  } catch (error) {
    console.error('Mailjet send error:', error?.statusCode, error?.message);
    return { success: false, error: error?.message || 'Failed to send email' };
  }
};
```

### Example Usage (Welcome Email on Signup)

```javascript
if (isProduction) {
    await sendEmail(
        newUser.email,
        "Welcome to Our App!",
        `<p>Hi ${newUser.username},</p><p>Welcome to our app! Please verify your email by clicking the link below:</p><a href="${backendUrl}/api/auth/verify-email?token=${emailVerificationToken}">Verify Email</a>`,
        `Hi ${newUser.username},\n\nWelcome to our app! Please verify your email by clicking the link below:\n${backendUrl}/api/auth/verify-email?token=${emailVerificationToken}`
    );
}
```

---

## 4. Quick Checklist

- [ ] Mailjet account created
- [ ] Domain added and DNS records (SPF/DKIM) verified
- [ ] Sender email address verified
- [ ] API Key & Secret Key added to `.env`
- [ ] `node-mailjet` and `dotenv` installed
- [ ] `mailjetClient.js` created
- [ ] `sendEmail` function integrated into controller(s)
- [ ] Tested in production mode (`isProduction` flag)

---

## 5. Notes

- Mailjet's free tier has a daily sending limit — check your plan on the [Mailjet pricing page](https://www.mailjet.com/pricing/) if you hit limits.
- For advanced use cases (templates, attachments, CC/BCC), Mailjet's `v3.1` Send API supports additional fields like `TemplateID`, `Attachments`, `Cc`, and `Bcc`.
- Monitor delivery status, bounces, and opens from the Mailjet dashboard under **Stats**.