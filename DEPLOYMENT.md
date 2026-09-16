# Azure Deployment Guide

This project deploys frontend and backend separately:

- Frontend -> Azure Static Web Apps
- Backend -> Azure Container Apps

## Workflows

- Frontend workflow: `.github/workflows/azure-static-web-apps.yml`
- Backend workflow: `.github/workflows/deploy.yml`

## Required GitHub Actions Variables

- `VITE_API_BASE_URL`  
  Example: `https://novaband-api.<region>.azurecontainerapps.io`
- `AZURE_RESOURCE_GROUP`
- `AZURE_CONTAINER_APP_NAME`
- `AZURE_CONTAINER_REGISTRY_NAME`
- `AZURE_CONTAINER_REGISTRY_LOGIN_SERVER`

## Required GitHub Actions Secrets

- `AZURE_STATIC_WEB_APPS_API_TOKEN` (or generated SWA token secret name)
- `AZURE_CLIENT_ID`
- `AZURE_TENANT_ID`
- `AZURE_SUBSCRIPTION_ID`

## Backend Runtime Environment (Azure Container App)

Set these in `novaband-api` container app environment variables:

- `ANTHROPIC_API_KEY`
- `LLM_PROVIDER` (`anthropic` or `azure_openai`)
- `DATABASE_URL`
- `CORS_ALLOW_ORIGINS`

If using Azure OpenAI provider, also set:

- `AZURE_OPENAI_API_KEY`
- `AZURE_OPENAI_ENDPOINT`
- `AZURE_OPENAI_DEPLOYMENT_NAME`

## Supabase `DATABASE_URL` Recommendation

For Azure Container Apps, use Supabase **Session Pooler** + SSL:

```env
DATABASE_URL=postgresql://postgres.<project-ref>:<password>@aws-0-eu-west-2.pooler.supabase.com:5432/postgres?sslmode=require
```

Notes:

- Keep `?sslmode=require`
- Prefer session pooler to avoid IPv6-only direct-host issues from Azure runtime

## CORS

Set `CORS_ALLOW_ORIGINS` to your frontend URL:

```env
CORS_ALLOW_ORIGINS=https://<your-static-web-app>.azurestaticapps.net
```

Multiple origins can be comma-separated.

## Moderation + Escalation Behavior

- Abuse is classified by the model on each user message.
- Strike counts are persisted in PostgreSQL (`sessions.strike_count`).
- At 2 strikes, session is escalated and closed.
- After escalation, backend always returns deterministic `SESSION_CLOSED_REPLY`.

## Validation Commands

### Backend health

```bash
curl https://novaband-api.proudsand-efd36542.eastus.azurecontainerapps.io/health
```

### Backend chat

```bash
curl -X POST https://novaband-api.proudsand-efd36542.eastus.azurecontainerapps.io/chat \
  -H "Content-Type: application/json" \
  -d '{"session_id":"prod-test","messages":[{"role":"user","content":"hi"}]}'
```

### CORS preflight

```bash
curl -i -X OPTIONS https://novaband-api.proudsand-efd36542.eastus.azurecontainerapps.io/chat \
  -H "Origin: https://agreeable-moss-08aec0f0f.2.azurestaticapps.net" \
  -H "Access-Control-Request-Method: POST" \
  -H "Access-Control-Request-Headers: content-type"
```

## Troubleshooting

- **Frontend calls wrong backend**
  - Confirm `VITE_API_BASE_URL` in GitHub Actions variables
  - Re-run frontend workflow

- **CORS errors**
  - Verify `CORS_ALLOW_ORIGINS` exactly matches frontend origin

- **Generic fallback reply from backend**
  - Check Container App logs for app exceptions
  - Validate `DATABASE_URL`, `LLM_PROVIDER`, provider keys

- **Moderation appears inconsistent**
  - Ensure tests use the same `session_id` for strike progression
  - Confirm latest backend revision has 100% traffic
