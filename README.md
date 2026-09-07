# Todo Microservices — Complete Azure Container Apps Deployment Guide

Note:- connection string url me ODBC 17 rakhna hai 18 ki jagah

## 1. Project Overview

This project contains four applications:

```text
Todo App
│
├── AddTaskTodoMicroservice
├── DeleteTaskTodoMicroservice
├── GetTasksTodoMicroservice
└── MicroTodoUI
```

### Final Architecture

```text
                         Internet
                            │
                            ▼
                    ┌───────────────┐
                    │    Browser    │
                    └───────┬───────┘
                            │
                            ▼
                  ┌───────────────────┐
                  │   MicroTodo UI    │
                  │ React + Nginx     │
                  │      :80          │
                  └───────┬───────────┘
                          │
             ┌────────────┼────────────┐
             │            │            │
             ▼            ▼            ▼
       ┌──────────┐ ┌────────────┐ ┌──────────┐
       │ GetTask  │ │ DeleteTask │ │ AddTask  │
       │   API    │ │    API     │ │   API    │
       │  :8000   │ │   :8000    │ │  :8000   │
       └────┬─────┘ └─────┬──────┘ └────┬─────┘
            │             │             │
            └─────────────┼─────────────┘
                          ▼
                   ┌──────────────┐
                   │  Azure SQL   │
                   │  Database    │
                   └──────────────┘
```

---

# 2. Prepare the Application

## Step 1 — Backend Structure

Each backend service should contain:

```text
app.py
requirements.txt
Dockerfile
```

The UI should contain:

```text
package.json
package-lock.json
src/
public/
.env
Dockerfile
```

The three backend services are:

- AddTaskTodoMicroservice
- DeleteTaskTodoMicroservice
- GetTasksTodoMicroservice

The frontend is:

- MicroTodoUI

---

# 3. Prepare Backend Code

## Step 2 — Use Environment Variable for SQL Connection

In each backend `app.py`, read the SQL connection string from an environment variable:

```python
connection_string = os.getenv("CONNECTION_STRING")

if not connection_string:
    raise RuntimeError("CONNECTION_STRING environment variable is not set")
```

Do not keep the SQL password hardcoded in application code.

---

# 4. Backend Dockerfiles

## Step 3 — Python and ODBC Configuration

Use this approach for all backend services:

```text
Python 3.9
Debian 12 / Bookworm
Microsoft ODBC Driver 17 for SQL Server
```

The backend container listens on:

```text
8000
```

Use the same working multi-stage Dockerfile pattern for the backend services.

---

# 5. Build Backend Docker Images

## Step 4 — Build AddTask Image

```powershell
cd .\AddTaskTodoMicroservice
docker build --no-cache -t sammyacr.azurecr.io/addtask:v1 .
```

Check:

```powershell
docker images
```

## Step 5 — Build DeleteTask Image

```powershell
cd ..\DeleteTaskTodoMicroservice
docker build --no-cache -t sammyacr.azurecr.io/deletetask:v1 .
```

## Step 6 — Build GetTask Image

```powershell
cd ..\GetTasksTodoMicroservice
docker build --no-cache -t sammyacr.azurecr.io/gettask:v1 .
```

---

# 6. Create Azure Resources

## Step 7 — Create Resource Group

Azure Portal:

**Resource Groups → Create**

Example:

```text
Resource Group:
todo-rg
```

Use the same Resource Group for the project resources.

---

# 7. Create Azure SQL

## Step 8 — Create SQL Server and Database

Azure Portal:

**SQL databases → Create**

Create the logical SQL Server and database.

Example:

```text
Server:
todo-sql-server

Database:
todo-db

Admin username:
sqladmin
```

Set a strong SQL administrator password.

## Step 9 — Configure SQL Networking

For the initial learning deployment:

- Enable Public network access.
- Allow Azure services to access the server.
- SQL Server uses port `1433`.

For production, use a more restrictive networking design.

---

# 8. Verify Azure SQL

## Step 10 — Database Verification

Connect to the SQL Database using Query Editor, SSMS, or another SQL client.

The application uses a `Tasks` table containing:

```text
ID
Title
Description
```

---

# 9. Create Azure Container Registry

## Step 11 — Create ACR

Azure Portal:

**Container Registries → Create**

Example:

```text
Registry name:
sammyacr
```

---

# 10. Login to ACR

## Step 12 — Azure Login

```powershell
az login
az acr login --name sammyacr
```

Expected:

```text
Login Succeeded
```

---

# 11. Push Backend Images to ACR

## Step 13 — Push AddTask

```powershell
docker push sammyacr.azurecr.io/addtask:v1
```

## Step 14 — Push DeleteTask

```powershell
docker push sammyacr.azurecr.io/deletetask:v1
```

## Step 15 — Push GetTask

```powershell
docker push sammyacr.azurecr.io/gettask:v1
```

ACR repositories should contain:

```text
addtask
deletetask
gettask
```

---

# 12. Create Container Apps Environment

## Step 16 — Create Container Apps Environment

Azure Portal:

**Container Apps → Create**

Create one Container Apps Environment.

Example:

```text
Container Apps Environment:
todo-env
```

Use this same environment for all four Container Apps.

```text
todo-env
│
├── addtask-api
├── delete-task-api
├── gettask-api
└── microtodo-ui
```

Do not create a separate environment for every application.

---

# 13. Create AddTask API Container App

## Step 17 — Create AddTask Container App

Create:

```text
Container App:
addtask-api
```

Use the existing:

```text
todo-env
```

Image:

```text
sammyacr.azurecr.io/addtask:v1
```

## Step 18 — Configure ACR Authentication

Use:

```text
Authentication:
Managed Identity
```

Use the system-assigned managed identity.

The identity needs:

```text
AcrPull
```

on the `sammyacr` registry.

## Step 19 — Configure AddTask Ingress

Enable ingress:

```text
Enabled
```

Traffic:

```text
External
```

Target port:

```text
8000
```

Protocol:

```text
HTTP
```

## Step 20 — Add AddTask SQL Secret

Container App → **Secrets**

Create:

```text
Name:
connection-string
```

Value:

```text
Driver={ODBC Driver 17 for SQL Server};Server=tcp:<server>.database.windows.net,1433;Database=<database>;Uid=<user>;Pwd=<password>;Encrypt=yes;TrustServerCertificate=yes;Connection Timeout=30;
```

Use your actual Azure SQL values.

## Step 21 — Add AddTask Environment Variable

Containers → **Environment variables**

```text
Name:
CONNECTION_STRING

Source:
Reference a secret

Secret:
connection-string
```

Flow:

```text
CONNECTION_STRING
        │
        ▼
connection-string secret
        │
        ▼
Azure SQL
```

## Step 22 — Deploy and Verify AddTask

Container App → **Overview**

Copy the **Application URL**.

Example:

```text
https://addtask-api.xxxxx.azurecontainerapps.io
```

Open the API URL and verify the API responds successfully.

Check container logs and confirm there are no SQL/ODBC connection errors.

---

# 14. Create DeleteTask API Container App

## Step 23 — Create DeleteTask Container App

Create:

```text
Container App:
delete-task-api
```

Use:

```text
todo-env
```

Image:

```text
sammyacr.azurecr.io/deletetask:v1
```

## Step 24 — Configure DeleteTask ACR Authentication

Use:

```text
Managed Identity
```

Give its managed identity:

```text
AcrPull
```

on:

```text
sammyacr
```

## Step 25 — Configure DeleteTask Ingress

Enable:

```text
External
```

Target port:

```text
8000
```

## Step 26 — Add DeleteTask SQL Secret

Create:

```text
Secret name:
connection-string
```

Use the same Azure SQL connection string:

```text
Driver={ODBC Driver 17 for SQL Server};Server=tcp:<server>.database.windows.net,1433;Database=<database>;Uid=<user>;Pwd=<password>;Encrypt=yes;TrustServerCertificate=yes;Connection Timeout=30;
```

## Step 27 — Add DeleteTask Environment Variable

```text
Name:
CONNECTION_STRING

Source:
Reference a secret

Secret:
connection-string
```

## Step 28 — Deploy and Verify DeleteTask

Container App → **Overview**

Copy its Application URL.

Check:

- Application is running.
- Container logs have no SQL connection error.
- API responds successfully.

---

# 15. Create GetTask API Container App

## Step 29 — Create GetTask Container App

Create:

```text
Container App:
gettask-api
```

Use:

```text
todo-env
```

Image:

```text
sammyacr.azurecr.io/gettask:v1
```

## Step 30 — Configure GetTask ACR Authentication

Use:

```text
Managed Identity
```

Give the managed identity:

```text
AcrPull
```

on:

```text
sammyacr
```

## Step 31 — Configure GetTask Ingress

Enable:

```text
External
```

Target port:

```text
8000
```

## Step 32 — Add GetTask SQL Secret

Create:

```text
Secret name:
connection-string
```

Value:

```text
Driver={ODBC Driver 17 for SQL Server};Server=tcp:<server>.database.windows.net,1433;Database=<database>;Uid=<user>;Pwd=<password>;Encrypt=yes;TrustServerCertificate=yes;Connection Timeout=30;
```

## Step 33 — Add GetTask Environment Variable

```text
Name:
CONNECTION_STRING

Source:
Reference a secret

Secret:
connection-string
```

## Step 34 — Deploy and Verify GetTask

Container App → **Overview**

Copy the Application URL.

Check the logs and verify the API can connect to Azure SQL and return data.

---

# 16. Prepare MicroTodo UI

## Step 35 — Update UI `.env`

Open:

```text
MicroTodoUI/.env
```

Replace old backend IP addresses with the three Container App Application URLs.

```env
REACT_APP_GET_TASKS_API_BASE_URL=https://GETTASK-APP-URL
REACT_APP_DELETE_TASK_API_BASE_URL=https://DELETE-TASK-APP-URL
REACT_APP_CREATE_TASK_API_BASE_URL=https://ADDTASK-APP-URL
```

Example:

```env
REACT_APP_GET_TASKS_API_BASE_URL=https://gettask-api.xxxxx.japaneast.azurecontainerapps.io
REACT_APP_DELETE_TASK_API_BASE_URL=https://delete-task-api.xxxxx.japaneast.azurecontainerapps.io
REACT_APP_CREATE_TASK_API_BASE_URL=https://addtask-api.xxxxx.japaneast.azurecontainerapps.io
```

### Mapping

```text
REACT_APP_GET_TASKS_API_BASE_URL
             ↓
        gettask-api

REACT_APP_DELETE_TASK_API_BASE_URL
             ↓
       delete-task-api

REACT_APP_CREATE_TASK_API_BASE_URL
             ↓
         addtask-api
```

Do not add `:8000` to these Container App URLs.

Correct:

```text
https://gettask-api.xxxxx.azurecontainerapps.io
```

Not:

```text
https://gettask-api.xxxxx.azurecontainerapps.io:8000
```

---

# 17. Prepare UI Dockerfile

## Step 36 — Multi-stage React + Nginx Dockerfile

```dockerfile
# ==========================
# Stage 1 - Build
# ==========================
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

RUN npm run build


# ==========================
# Stage 2 - Runtime
# ==========================
FROM nginx:alpine

COPY --from=builder /app/build /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

The React UI is served by Nginx on port:

```text
80
```

---

# 18. Build UI Docker Image

## Step 37 — Build UI Image

Go to:

```text
MicroTodoUI
```

Run:

```powershell
docker build --no-cache -t sammyacr.azurecr.io/microtodo-ui:v1 .
```

Check:

```powershell
docker images | findstr microtodo
```

Expected:

```text
sammyacr.azurecr.io/microtodo-ui    v1
```

---

# 19. Push UI Image to ACR

## Step 38 — Push UI Image

```powershell
docker push sammyacr.azurecr.io/microtodo-ui:v1
```

ACR should now contain:

```text
Repositories
│
├── addtask
├── deletetask
├── gettask
└── microtodo-ui
```

---

# 20. Create UI Container App

## Step 39 — Create MicroTodo UI Container App

Azure Portal:

**Container Apps → Create**

Create:

```text
Container App:
microtodo-ui
```

Use:

```text
todo-env
```

Image:

```text
sammyacr.azurecr.io/microtodo-ui:v1
```

---

# 21. Configure UI ACR Authentication

## Step 40

Use:

```text
Managed Identity
```

Use system-assigned identity.

Give it:

```text
AcrPull
```

on:

```text
sammyacr
```

---

# 22. Configure UI Ingress

## Step 41

Enable:

```text
Ingress:
External
```

Target port:

```text
80
```

Important:

```text
React/Nginx UI → Port 80
FastAPI APIs → Port 8000
```

---

# 23. Deploy UI

## Step 42 — Create Container App

Click:

**Review + create → Create**

Wait for deployment to finish.

Then:

Container App → **Overview**

Copy:

```text
Application URL
```

Example:

```text
https://microtodo-ui.xxxxx.japaneast.azurecontainerapps.io
```

Open this URL in the browser.

---

# 24. Final Application Testing

## Step 43 — Test Get Tasks

Open the UI and verify existing tasks are displayed.

Flow:

```text
Browser
   ↓
MicroTodo UI
   ↓
gettask-api
   ↓
Azure SQL
```

## Step 44 — Test Add Task

Create a new task.

Flow:

```text
Browser
   ↓
MicroTodo UI
   ↓
addtask-api
   ↓
Azure SQL
```

Verify the new task appears.

## Step 45 — Test Delete Task

Delete a task.

Flow:

```text
Browser
   ↓
MicroTodo UI
   ↓
delete-task-api
   ↓
Azure SQL
```

Verify the task is removed.

---

# 25. Troubleshooting

## UI opens but data does not appear

Open browser:

```text
F12 → Console
```

and:

```text
F12 → Network
```

Check which API URL the React application is calling.

It should be the Container App URL:

```text
https://gettask-api....azurecontainerapps.io
```

and not an old IP address.

If an old IP is being called, update `.env` and rebuild the UI image.

---

## `argument 1 must be a string or unicode object`

This usually indicates that the backend did not receive a valid `CONNECTION_STRING`.

Check:

```text
Secret:
connection-string
```

and:

```text
Environment variable:
CONNECTION_STRING
Source:
Reference a secret
Secret:
connection-string
```

---

## ODBC Driver error

For this project, the connection string should use:

```text
Driver={ODBC Driver 17 for SQL Server};
```

The backend image should have:

```text
Microsoft ODBC Driver 17
```

installed.

---

# 26. Final Azure Resource Structure

```text
Resource Group
│
├── Azure SQL Server
│     └── SQL Database
│
├── Azure Container Registry
│     ├── addtask:v1
│     ├── deletetask:v1
│     ├── gettask:v1
│     └── microtodo-ui:v1
│
└── Container Apps Environment
      │
      ├── addtask-api
      │     ├── External :8000
      │     ├── Secret: connection-string
      │     └── ENV: CONNECTION_STRING
      │
      ├── delete-task-api
      │     ├── External :8000
      │     ├── Secret: connection-string
      │     └── ENV: CONNECTION_STRING
      │
      ├── gettask-api
      │     ├── External :8000
      │     ├── Secret: connection-string
      │     └── ENV: CONNECTION_STRING
      │
      └── microtodo-ui
            └── External :80
```

---

# 27. Complete Deployment Order

```text
1. Prepare application code
2. Configure backend CONNECTION_STRING
3. Prepare backend Dockerfiles
4. Build AddTask image
5. Build DeleteTask image
6. Build GetTask image
7. Create Resource Group
8. Create Azure SQL Server
9. Create Azure SQL Database
10. Configure SQL networking
11. Create Azure Container Registry
12. Login to Azure
13. Login to ACR
14. Push AddTask image
15. Push DeleteTask image
16. Push GetTask image
17. Create Container Apps Environment
18. Create AddTask Container App
19. Configure AddTask managed identity
20. Give AddTask AcrPull
21. Configure AddTask external ingress :8000
22. Add AddTask SQL secret
23. Add AddTask CONNECTION_STRING environment variable
24. Deploy and verify AddTask
25. Create DeleteTask Container App
26. Configure DeleteTask managed identity
27. Give DeleteTask AcrPull
28. Configure DeleteTask external ingress :8000
29. Add DeleteTask SQL secret
30. Add DeleteTask CONNECTION_STRING environment variable
31. Deploy and verify DeleteTask
32. Create GetTask Container App
33. Configure GetTask managed identity
34. Give GetTask AcrPull
35. Configure GetTask external ingress :8000
36. Add GetTask SQL secret
37. Add GetTask CONNECTION_STRING environment variable
38. Deploy and verify GetTask
39. Update MicroTodoUI .env
40. Prepare UI Dockerfile
41. Build UI image
42. Push UI image to ACR
43. Create MicroTodo UI Container App
44. Configure UI managed identity
45. Give UI AcrPull
46. Configure UI external ingress :80
47. Deploy UI
48. Open UI Application URL
49. Test Get Task
50. Test Add Task
51. Test Delete Task
```

---

# 28. Final End-to-End Flow

```text
Developer
    │
    ▼
Docker Build
    │
    ▼
Docker Images
    │
    ▼
Azure Container Registry
    │
    ▼
Azure Container Apps
    │
    ├──────────────┐
    │              │
    ▼              ▼
FastAPI APIs     React UI
    │              │
    └───────┬──────┘
            ▼
        Azure SQL
```

The final user request flow is:

```text
Browser
   │
   │ HTTPS
   ▼
MicroTodo UI
   │
   ├── HTTPS → GetTask API
   │
   ├── HTTPS → DeleteTask API
   │
   └── HTTPS → AddTask API
                    │
                    ▼
                Azure SQL
```
