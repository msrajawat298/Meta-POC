# Next.js Frontend + Node.js Backend Microservices on Azure

## Overview
This plan describes how to build a secure frontend/backend architecture using:
- Next.js for the frontend
- Node.js microservices for backend APIs
- Azure for deployment, secrets, and hosting

The main goal is to keep Meta/Facebook tokens and any secrets on the backend, never exposed to the browser.

---

## Architecture

1. Next.js frontend
   - Hosted as a static app or SSR app
   - Calls backend APIs only
   - Does not contain Facebook access tokens

2. Node.js backend microservices
   - One or more services that call Meta Graph API
   - Runs on Azure (App Service, Container Apps, or Functions)
   - Uses environment variables / Azure Key Vault for secrets

3. Azure services
   - Azure Static Web Apps / Azure App Service for frontend
   - Azure App Service / Azure Container Apps / Azure Functions for backend
   - Azure Key Vault for token storage
   - Azure App Configuration or App Service settings for environment variables

---

## Deployment pattern

1. Local development
   - Frontend: `npm run dev` (Next.js)
   - Backend: `node server.js` or local microservice startup
   - Use `.env.local` and `.env` for local secrets

2. Build and deploy
   - Frontend deploys to Azure Static Web Apps or App Service
   - Backend deploys to Azure microservice hosts
   - Backend services expose secure API endpoints like `/api/facebook-media`

3. Runtime flow
   - Browser -> Next.js UI
   - UI -> backend API
   - Backend -> Meta Graph API using `PAGE_ACCESS_TOKEN`
   - Backend returns sanitized JSON to UI

---

## Step-by-step implementation

### 1. Define service boundaries
- Frontend service: Next.js app
- Backend service: Node.js microservice for Meta API calls
- Optional auth service: if you need user login and token management

### 2. Create backend API

Example folder structure:
```
/Meta-POC
  /frontend
  /backend
    /facebook-service
      server.js
      package.json
      .env
```

Example backend endpoint:
```js
// backend/facebook-service/server.js
import express from 'express';
import fetch from 'node-fetch';

const app = express();
const port = process.env.PORT || 4000;

app.get('/facebook-media', async (req, res) => {
  const pageToken = process.env.FB_PAGE_ACCESS_TOKEN;
  if (!pageToken) {
    return res.status(500).json({ error: 'Missing page token' });
  }

  const url = new URL('https://graph.facebook.com/v18.0/17841411615424388/media');
  url.searchParams.set('fields', 'id,caption,media_type,media_url,thumbnail_url,permalink,timestamp');
  url.searchParams.set('access_token', pageToken);

  const response = await fetch(url);
  const data = await response.json();
  if (!response.ok) {
    return res.status(response.status).json(data);
  }
  res.json(data);
});

app.listen(port, () => {
  console.log(`Facebook service running on port ${port}`);
});
```

### 3. Keep tokens secure
- Do not put tokens in client code.
- Store long-lived `PAGE_ACCESS_TOKEN` in Azure App Settings or Key Vault.
- Use `FB_PAGE_ACCESS_TOKEN` only in backend service.
- If you need to refresh the token every 45 days, update the backend secret using a secure process.

### 4. Connect frontend to backend

In Next.js, call your backend route from client code:

```js
// frontend/components/MediaList.js
import { useEffect, useState } from 'react';

export default function MediaList() {
  const [media, setMedia] = useState(null);

  useEffect(() => {
    fetch('/api/facebook-media')
      .then((res) => res.json())
      .then(setMedia)
      .catch(console.error);
  }, []);

  if (!media) return <div>Loading...</div>;
  return <pre>{JSON.stringify(media, null, 2)}</pre>;
}
```

If frontend and backend are deployed separately, use the backend service URL directly.

### 5. Use backend routing if frontend and backend share domain

If you want a single domain, use Next.js API routes or frontend rewrites.

Example `pages/api/facebook-media.js`:
```js
export default async function handler(req, res) {
  const apiUrl = process.env.BACKEND_FACEBOOK_SERVICE_URL;
  const response = await fetch(`${apiUrl}/facebook-media`);
  const data = await response.json();
  res.status(response.status).json(data);
}
```

### 6. Deploy to Azure

#### Option A: Azure Static Web Apps for frontend
- Deploy Next.js frontend as a Static Web App
- Use Azure App Service or Container Apps for backend
- Configure frontend to call backend URL from environment

#### Option B: Azure App Service for frontend and backend
- Deploy frontend to one App Service
- Deploy backend to another App Service or App Service plan
- Use App Service settings for environment variables

#### Option C: Azure Container Apps or Kubernetes
- Containerize frontend and backend
- Deploy both to Azure Container Apps or Azure Kubernetes Service
- Use secure environment variables / managed identity

### 7. Setup Azure secrets and configuration
- Use Azure Portal or CLI to set App Settings:
  - `FB_PAGE_ACCESS_TOKEN`
  - `FB_APP_ID`
  - `FB_APP_SECRET`
  - `BACKEND_FACEBOOK_SERVICE_URL`
- Optionally use Azure Key Vault and reference secrets from App Service

### 8. Add CI/CD
- Use GitHub Actions or Azure DevOps
- Build frontend separately
- Build backend separately
- Deploy to Azure App Service or Static Web Apps
- Ensure secrets are only stored in Azure, not source control

Example GitHub Actions outline:
- Checkout code
- Install dependencies
- Build frontend
- Build backend
- Deploy frontend to Azure Static Web Apps or App Service
- Deploy backend to Azure App Service

---

## Recommended folder layout

```
/Meta-POC
  /frontend
    package.json
    next.config.js
    pages/ or app/
  /backend
    /facebook-service
      package.json
      server.js
      Dockerfile? (optional)
```

---

## Security rules

- Never expose `PAGE_ACCESS_TOKEN` or `USER_ACCESS_TOKEN` in browser code.
- Do not commit `.env` with secrets.
- Use `gitignore` for local env files.
- Use HTTPS for all browser-to-backend and backend-to-Facebook calls.
- Use Azure App Service authentication if needed.

---

## Token refresh process

1. Generate a long-lived user token.
2. Exchange for page access token if needed.
3. Store the page token in Azure App Settings or Key Vault.
4. Rotate tokens every 45 days.
5. Update backend secret value when token changes.

---

## Summary

- Build Next.js frontend separately from Node backend.
- Keep Meta tokens on backend only.
- Deploy frontend and backend as Azure services.
- Use App Settings / Key Vault for secrets.
- Use secure API calls from frontend to backend, not directly to Facebook.


Backend:
// pages/api/facebook-media.js
export default async function handler(req, res) {
  const pageToken = process.env.FB_PAGE_ACCESS_TOKEN;
  if (!pageToken) {
    return res.status(500).json({ error: 'Missing page token' });
  }

  const url = new URL('https://graph.facebook.com/v18.0/17841411615424388/media');
  url.searchParams.set('fields', 'id,caption,media_type,media_url,thumbnail_url,permalink,timestamp');
  url.searchParams.set('access_token', pageToken);

  const response = await fetch(url.toString());
  const data = await response.json();

  if (!response.ok) {
    return res.status(response.status).json(data);
  }

  return res.status(200).json(data);
}


Client Side:

// components/MediaList.js
import { useEffect, useState } from 'react';

export default function MediaList() {
  const [media, setMedia] = useState(null);

  useEffect(() => {
    fetch('/api/facebook-media')
      .then((res) => res.json())
      .then(setMedia)
      .catch(console.error);
  }, []);

  if (!media) return <div>Loading...</div>;
  return <pre>{JSON.stringify(media, null, 2)}</pre>;
}

