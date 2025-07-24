# Adding SSL (HTTPS) to Your Project

## Background

Security is a core requirement for any modern web application, especially those that handle user input or personal data.  
This guide will walk you through setting up SSL (Secure Sockets Layer), more accurately called TLS, for both backend and frontend apps in a local development environment.

Before you start, you will research what SSL is, why it is necessary, and the impact it has on real-world web applications.

## Research Activity: Why SSL?

Spend 10–15 minutes researching the following questions:

- What is SSL/TLS?  
- How does HTTPS differ from HTTP?
- Why is SSL necessary in web applications?
- What could happen if a web application does not use SSL?
- Find one real-world example (news or blog) of a security incident involving missing or misconfigured SSL.

Write a short summary (4–6 sentences) in your own words. Add this write up into your repo as well.

Suggested links:
- https://developer.mozilla.org/en-US/docs/Web/Security/HTTP_strict_transport_security
- https://letsencrypt.org/docs/why-https/

# Setting Up SSL for Local Development

## Overview

This guide explains how to create a self-signed SSL certificate and configure it for local development, so you can run your backend and frontend over HTTPS.

### 1. Create the OpenSSL Config File

In your backend project folder, create a folder named `ssl` if it doesn’t exist.

Inside `ssl`, create a file named `openssl.cnf` and paste the following:
```
[req]
distinguished_name = req_distinguished_name
x509_extensions = v3_req
prompt = no

[req_distinguished_name]
C = ZA
ST = KZN
L = Durban
O = IIE VC
OU = DN
CN = localhost

[v3_req]
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = localhost
```

### 2. Generate the Certificate and Key Using Git Bash

Open Git Bash and run 
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ssl/key.pem -out ssl/cert.pem -config ssl/openssl.cnf -extensions v3_req
```
You should now have `key.pem` and `cert.pem` in `backend/ssl`.

### 3. Copy Certificate and Key to the Frontend

In your frontend project, create an `ssl` folder.

Copy `key.pem` and `cert.pem` from `backend/ssl/` to `frontend/ssl/`.

You can use the same key and certificate if the api and client are going to the same domain. In this case, localhost. Otherwise, you will need to generate two separate certificates.

### 4. Update Your Code to Use SSL

#### Node.js Backend (server.js)

Update `server.js` to include:

```js
const mongoose = require('mongoose');
const app = require('./app');
const https = require('https');
const fs = require('fs');
require('dotenv').config();

const PORT = process.env.PORT || 5000;

const options = {
  key: fs.readFileSync('ssl/key.pem'),
  cert: fs.readFileSync('ssl/cert.pem'),
};

https.createServer(options, app).listen(PORT, () => {
  console.log('Server running at https://localhost:' + PORT);
});
```

#### React Front-end
Update `vite.config.js` to include:
```js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import fs from 'fs';

export default defineConfig({
  plugins: [react()],
  server: {
    https: {
      key: fs.readFileSync('ssl/key.pem'),
      cert: fs.readFileSync('ssl/cert.pem'),
    }
  }
});
```

### 5. Testing With a Self-Signed, Untrusted Certificate
Star
t your backend and frontend.

Visit:
- https://localhost:5000/
- https://localhost:5173/

You’ll see a browser security warning.
This is expected—your certificate is self-signed and not trusted.

### 6. Why Trust a Certificate?
A trusted certificate removes browser warnings and simulates a real HTTPS connection.
For production, always use a trusted CA certificate.

### 7. Making the Certificate Trusted (Windows)
To trust your cert and remove warnings:
- Press Win + R, type certmgr.msc, and press Enter.
- Expand Trusted Root Certification Authorities and select Certificates.
- Right-click in the right pane, select All Tasks > Import...
- Import ssl/cert.pem as a certificate file.
- Make sure the certificate store is Trusted Root Certification Authorities.
- Finish the wizard, accept warnings, and restart your browser.

### 8. Test Again
You should now see a padlock icon and no warning when visiting your app.

### 9. Reminders
- Never use self-signed certificates in production.
- Do not share your private key.
- Add ssl/ to .gitignore in both projects.

## Git
Commit and push your updates to GitHub.

## A note about SSL in Production

Self-signed certificates are for development only. For production, use a certificate from a trusted certificate authority such as Let's Encrypt, your hosting provider, or another recognized CA.

In most production setups, SSL is handled by your web server (e.g., NGINX, Apache) or cloud hosting platform, not directly in your application code.  
Research how SSL is typically managed for your chosen deployment stack and summarise your findings.



## Reflection

Briefly reflect on the following after completing the steps above:

- Did adding SSL/HTTPS affect any part of your application?
- Were there any challenges in trusting the certificate or configuring your stack?
- What steps would be different when preparing for production deployment?

Add your reflection as a section in `ssl_research.md`.

---
