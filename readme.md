📝 Project Deployment & Troubleshooting Report
🔹 Project Overview

Architecture: Microservices (8 services + API Gateway + Frontend).

Database: MySQL (local, schema imported from dump file).

Deployment: Dockerized & containerized with GitHub Actions.

CI/CD:

GitHub Actions → Build & push Docker images.

GitLab → Container Registry (images pushed & pulled).

GitLab Runner → Self-hosted runner on Contabo for direct deployment.

Code Quality: SonarQube integrated.

New Requirement: LinkedIn integration for project notifications/updates.

🔹 Key Issues & Solutions
1️⃣ Nginx Issues

Error:

nginx: [emerg] open() "/etc/nginx/sites-enabled/heavenly-hub-apigateway.conf" failed


Root Cause: Missing config file in sites-enabled.
Solution:

Create config under /etc/nginx/sites-available/heavenly-hub-apigateway.conf.

Add symlink:

sudo ln -s /etc/nginx/sites-available/heavenly-hub-apigateway.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl restart nginx

2️⃣ MySQL Database Setup

Problem: Microservices need a common schema.
Solution:

Dump schema from repo:

mysql -u admin -p < schema_dump.sql


Create separate DBs if each service requires isolation.

Update .env with proper host/port.

3️⃣ Dockerization

Problem: Some services not starting due to missing .env.
Solution:

Added .env support in each service.

Configured docker-compose.yml with healthchecks and depends_on for service order.

4️⃣ GitHub Actions CI/CD

Pipeline Steps:

Checkout code.

Build Docker image per service.

Push to GitLab Container Registry.

Trigger Contabo GitLab Runner to deploy using docker-compose up -d.

5️⃣ SonarQube Integration

Problem: Quality gate not applied.
Solution:

Added SonarQube step in GitHub Actions with sonar-scanner.

Configured Sonar token as GitHub Secret.

6️⃣ LinkedIn Integration (New)

Purpose: Post updates on build/deployment status to LinkedIn.

Approach:

Use LinkedIn API + GitHub Actions workflow.

Steps:

Create a LinkedIn App → Get Client ID & Secret.

Get OAuth2 Access Token.

In GitHub Actions, add LINKEDIN_ACCESS_TOKEN as secret.

Add a job step:

- name: Post update to LinkedIn
  run: |
    curl -X POST https://api.linkedin.com/v2/ugcPosts \
    -H "Authorization: Bearer ${{ secrets.LINKEDIN_ACCESS_TOKEN }}" \
    -H "Content-Type: application/json" \
    -d '{
      "author": "urn:li:person:YOUR_ID",
      "lifecycleState": "PUBLISHED",
      "specificContent": {
        "com.linkedin.ugc.ShareContent": {
          "shareCommentary": {
            "text": "🚀 New deployment of Microservices Project completed successfully via GitHub Actions & Docker!"
          },
          "shareMediaCategory": "NONE"
        }
      },
      "visibility": {
        "com.linkedin.ugc.MemberNetworkVisibility": "CONNECTIONS"
      }
    }'


This ensures every successful deployment posts a LinkedIn update 🚀.

🔹 Final Architecture (After Fixes & Integration)

Frontend → API Gateway → 8 Microservices.

MySQL (single instance, imported dump).

CI/CD: GitHub Actions + GitLab Runner (Contabo).

Monitoring: SonarQube + GitHub build logs.

Notifications: LinkedIn API integration.

Debugging Report – API Gateway (Nginx + Backend)
1. Frontend Login Issue

When attempting to log in with super-admin@ebutler.com, the frontend makes a request to:

https://heavenly-hub.veroke.com/apigateway/user/api/admin/check-email


The request fails with 405 Method Not Allowed.

2. Nginx Config Problem (Proxy Pass IP)

Current config:

proxy_pass http://127.0.0.0:3000/;


Issue: 127.0.0.0 is not a valid loopback IP.

Correct value:

proxy_pass http://127.0.0.1:3000/;

3. CORS / Preflight Requests

Error may also be related to CORS (Cross-Origin Resource Sharing).

Frontend likely sends OPTIONS requests before POST.

Nginx config must handle OPTIONS to avoid 405 Method Not Allowed.

✅ Fix: Add inside /apigateway/ block:

if ($request_method = OPTIONS) {
    add_header Access-Control-Allow-Origin *;
    add_header Access-Control-Allow-Methods "GET, POST, OPTIONS";
    add_header Access-Control-Allow-Headers "Authorization,Content-Type";
    return 204;
}

4. SSL Certificate Domain Mismatch

Browser request domain: heavenly-hub.veroke.com

SSL config uses: api-heavenly-hub.veroxm.com

⚠️ Issue: Domain mismatch may break HTTPS or cause requests to fail.
👉 Action: Generate SSL for the exact domain in use (heavenly-hub.veroke.com) with Certbot.

5. Backend Listening Port

Nginx proxies to port 3000.

Need to confirm backend is listening on 127.0.0.1:3000.

Check with:

ss -tulnp | grep 3000

✅ Immediate Action Plan

Fix proxy_pass IP → change 127.0.0.0 → 127.0.0.1.

Handle CORS preflight requests in Nginx config.

Verify backend service is running and listening on 127.0.0.1:3000.

Check SSL certificate → must match heavenly-hub.veroke.com.

Reload Nginx after changes:

sudo nginx -t
sudo systemctl reload nginx
roject Debugging & Solution Report
1. Project Setup & Issues Faced

Running a microservice project using GitHub Actions (CI/CD).

Containers deployed on server with MySQL already running.

New MySQL user admin created, database configured inside container.

Containers build & run successfully, logs show MySQL connected.

But accessing services via browser or curl returned:

{"status":"error","statusCode":405,"message":"Method Not Allowed. Please use the correct method for access","data":{},"error":405}

2. Identified Errors & Fixes
🔹 Error 1: Nginx Reverse Proxy Misconfiguration

Initial config:

proxy_pass http://127.0.0.0:3000/;


❌ Wrong loopback IP (127.0.0.0 instead of 127.0.0.1).

✅ Fixed config:

server {
    server_name heavenly-hub.veroke.com;

    location /apigateway/ {
        proxy_pass http://127.0.0.1:3009/;  
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}


Restarted Nginx:

sudo nginx -t
sudo systemctl restart nginx

🔹 Error 2: MySQL Connection Issues

Needed to confirm DB connection inside container:

docker exec -it <container_name> mysql -h 172.17.0.1 -P 3307 -u admin -p


Logs confirmed connection OK.

🔹 Error 3: 405 Method Not Allowed

Service required correct HTTP method (e.g., POST vs GET).

Example test:

curl -X POST http://heavenly-hub.veroke.com/apigateway/endpoint \
     -H "Content-Type: application/json" \
     -d '{"key":"value"}'


✅ Solution: Verify API docs → Use correct request methods.

🔹 Error 4: Missing Tools inside Container

Tried running docker and mysql inside Node.js container → commands not found.

✅ Solution: These tools are host-level, not inside app containers.

Run Docker commands on host server.

Use docker exec only for service-level debugging.

🔹 Error 5: Downloading AnyDesk on Linux (CentOS/RHEL 7)

Initial attempt failed with missing package version.

✅ Solution: Download latest from official site:
AnyDesk Linux Download

Install:

sudo rpm -i anydesk-7.0.2-1.el7.x86_64.rpm
anydesk

3. New Learnings from Debugging

Nginx reverse proxy needs correct loopback (127.0.0.1 not 127.0.0.0).

Always check API HTTP method (405 = wrong method).

Docker CLI tools only exist on host machine, not inside app containers.

MySQL connection must be tested with correct host, port, user, password.

Use official vendor pages for downloading Linux packages.

4. Final Status

✅ Nginx fixed → correct proxy pass.
✅ MySQL connectivity confirmed.
✅ 405 error solved by correct HTTP method usage.
✅ AnyDesk installed successfully on server.
