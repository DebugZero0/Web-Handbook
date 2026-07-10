# 🔐 Google OAuth Setup for Nodemailer (Node.js)

> A complete step-by-step guide to configure **Google OAuth 2.0** for sending emails using **Nodemailer** in a Node.js application.

---

## 📌 Prerequisites

Before starting, make sure you have:

- ✅ A Google Account
- ✅ Node.js installed
- ✅ A Google Cloud Project
- ✅ npm or yarn

---

# 🚀 Step 1: Create a Google Cloud Project

1. Visit the **Google Cloud Console**
   - https://console.cloud.google.com/

2. Click on

   ```
   Select Project → New Project
   ```

3. Enter

   - Project Name
   - Organization (Optional)

4. Click

   ```
   Create
   ```

---

# 🔑 Step 2: Enable Gmail API

1. Open

   https://console.cloud.google.com/apis/library

2. Search for

   ```
   Gmail API
   ```

3. Click on the Gmail API.

   ```
   Enable it
   ```

---

# 🔒 Step 3: Configure OAuth Consent Screen

Navigate to

https://console.cloud.google.com/apis/credentials/consent

### Fill in:

- App Name
- User Support Email(External)
- Developer Contact Email

Click

```
Save and Continue
```

---

# 👤 Step 4: Add Test Users

If your app is in **Testing Mode**

Go to

```
OAuth Consent Screen
```

Add your Gmail account under

```
Test Users 
```

Save.

---

# 🔑 Step 5: Create OAuth Credentials

Open

https://console.cloud.google.com/apis/credentials

Click

```
Create Credentials
```

Choose

```
OAuth Client ID
```

Application Type

```
Web Application
```

Example Name

```
web-client 1
```

---

## Authorized Redirect URI

Add

```
https://localhost
```
https://developers.google.com/oauthplayground
```

Click

```
---

# 📋 Step 6: Copy Credentials

You'll receive:

- Client ID
- Client Secret

Save both securely.

---

# 🎯 Step 7: Generate Refresh Token

Open

https://developers.google.com/oauthplayground/

---

## Click ⚙ Settings

Enable

```
Use your own OAuth credentials
```

Paste

- Client ID
- Client Secret

Save.

---

## Select Gmail API

Choose

```
Gmail API v1
```

Select

```
https://mail.google.com/
```

or

```
https://www.googleapis.com/auth/gmail.send
```

Click

```
Authorize APIs
```

Login with your Gmail account.

---

## Exchange Authorization Code

Click

```
Exchange authorization code for tokens
```

Copy the

```
Refresh Token From Step 2
```
Ignore the Bad Request error if it appears.
This token usually remains valid until revoked.

Add the refresh token to your `.env` file.
Add the google_user email to your `.env` file as well.(Make sure to add same email as added in test users)

---

# 📦 Step 8: Install Packages

```bash
npm install nodemailer googleapis dotenv
```

---

# 📁 Step 9: Create Environment Variables

Create a

```
.env
```

file.

```env
EMAIL_USER=your-email@gmail.com

GOOGLE_CLIENT_ID=xxxxxxxxxxxxxxxxxxxxxxxx

GOOGLE_CLIENT_SECRET=xxxxxxxxxxxxxxxxxxxx

GOOGLE_REFRESH_TOKEN=xxxxxxxxxxxxxxxxxxxx

REDIRECT_URI=https://developers.google.com/oauthplayground
```

> **Never commit your `.env` file to GitHub.**

---

# 💻 Step 10: Configure Nodemailer

```javascript
const nodemailer = require("nodemailer");
const { google } = require("googleapis");

const OAuth2 = google.auth.OAuth2;

const oauth2Client = new OAuth2(
  process.env.GOOGLE_CLIENT_ID,
  process.env.GOOGLE_CLIENT_SECRET,
  process.env.REDIRECT_URI
);

oauth2Client.setCredentials({
  refresh_token: process.env.GOOGLE_REFRESH_TOKEN,
});

async function sendMail() {
  const accessToken = await oauth2Client.getAccessToken();

  const transporter = nodemailer.createTransport({
    service: "gmail",
    auth: {
      type: "OAuth2",
      user: process.env.EMAIL_USER,
      clientId: process.env.GOOGLE_CLIENT_ID,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET,
      refreshToken: process.env.GOOGLE_REFRESH_TOKEN,
      accessToken: accessToken.token,
    },
  });

  await transporter.sendMail({
    from: process.env.EMAIL_USER,
    to: "receiver@example.com",
    subject: "Hello 👋",
    text: "Email sent using Google OAuth2",
  });

  console.log("Email Sent Successfully!");
}

sendMail();
```

---

# 📂 Project Structure

```
project/
│
├── node_modules/
├── .env
├── package.json
├── package-lock.json
├── app.js
└── README.md
```

---

# 🛡 Security Tips

- Never expose your Client Secret.
- Never upload `.env` to GitHub.
- Add `.env` to `.gitignore`.
- Rotate credentials if they are accidentally leaked.
- Revoke refresh tokens if you suspect compromise.

---

# 📚 Useful Links

| Resource | Link |
|----------|------|
| Google Cloud Console | https://console.cloud.google.com/ |
| Gmail API | https://console.cloud.google.com/apis/library/gmail.googleapis.com |
| OAuth Credentials | https://console.cloud.google.com/apis/credentials |
| OAuth Consent Screen | https://console.cloud.google.com/apis/credentials/consent |
| OAuth Playground | https://developers.google.com/oauthplayground/ |
| Nodemailer Documentation | https://nodemailer.com/ |
| Google OAuth Documentation | https://developers.google.com/identity/protocols/oauth2 |

---

# ✅ Required Credentials Checklist

- [ ] Gmail API Enabled
- [ ] OAuth Consent Screen Configured
- [ ] OAuth Client ID
- [ ] OAuth Client Secret
- [ ] Redirect URI Added
- [ ] Refresh Token Generated
- [ ] `.env` Configured
- [ ] Nodemailer Installed
- [ ] Email Successfully Sent

---

# 🎉 Done!

Your Node.js application is now configured to send emails securely using **Google OAuth 2.0** with **Nodemailer**, eliminating the need for less secure app passwords.

IF new Refresh is to be generated,copy the exixting client ID and secret and repeat Step 7.

Happy Coding! 🚀