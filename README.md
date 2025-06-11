# 🚢 Wanderlust – Dockerized Setup

This repository containerizes the **Wanderlust** fullstack blog application using Docker and Docker Compose. It includes services for the frontend (Vite), backend (Node.js + Express), and MongoDB.

---

## 🧱 1. Project Structure (Dockerized)

```
wanderlust/
├── backend/
│   ├── Dockerfile
│   ├── .env.sample
│   └── data/
│       └── sample_posts.json
├── frontend/
│   ├── Dockerfile
│   ├── .env.sample
└── docker-compose.yml
```

---

## 🐳 2. Dockerfile Breakdown

### Backend `backend/Dockerfile`

```dockerfile
FROM node:21 AS builder

WORKDIR /app
COPY package*.json ./
RUN npm i
COPY . .

FROM node:21-slim
WORKDIR /app
COPY --from=builder /app ./
COPY .env.sample .env.local
EXPOSE 5000
CMD ["npm", "start"]
```

✅ **Explanation**:
- Stage 1 installs dependencies and copies source files.
- Stage 2 runs the lightweight image for runtime.
- Exposes port `5000`.
- Copies `.env.sample` to `.env.local` for configuration inside the container.

---

### Frontend `frontend/Dockerfile`

```dockerfile
FROM node:21 AS builder

WORKDIR /app
COPY package*.json ./
RUN npm i
COPY . .

FROM node:21-slim
WORKDIR /app
COPY --from=builder /app ./
COPY .env.sample .env.local
CMD ["npm", "run", "dev", "--", "--host"]
```

✅ **Explanation**:
- Uses Vite's dev server.
- `--host` allows access from outside the container (e.g., your browser hitting the EC2 instance).

---

## 🧩 3. Docker Compose Overview

### `docker-compose.yml`

```yaml
version: '3.8'

services:
  mongodb:
    container_name: mongo
    image: mongo:latest
    volumes:
      - ./backend/data:/data
    ports:
      - "27017:27017"

  backend:
    container_name: backend
    build: ./backend
    env_file:
      - ./backend/.env.sample
    ports:
      - "5000:5000"
    depends_on:
      - mongodb

  frontend:
    container_name: frontend
    build: ./frontend
    env_file:
      - ./frontend/.env.sample
    ports:
      - "5173:5173"
    depends_on:
      - backend

volumes:
  data:
```

✅ **Compose Explanation**:
- Spins up **MongoDB**, **backend**, and **frontend** as isolated services.
- Uses volume to persist MongoDB data.
- All services depend on the proper start order (`depends_on`).

---

## 📦 4. Volume Explanation

```yaml
volumes:
  - ./backend/data:/data
```

✅ Binds your host’s `./backend/data` to MongoDB’s internal `/data` directory:
- Enables import of seed data.
- Data persists across restarts.

---

## ⚙️ 5. Build and Run

### 🔨 Start All Containers
```bash
docker-compose up --build
```

Rebuilds images and runs all services.

### ⏹️ Stop & Remove Containers
```bash
docker-compose down
```

Stops and cleans up all services.

---

## 🧬 6. Import Sample Data into MongoDB

After the containers are up:

### 🛠️ Exec into the MongoDB container:
```bash
docker exec -it mongo bash
```

### 📥 Run the import command:
```bash
mongoimport --db wanderlust --collection posts --file ./data/sample_posts.json --jsonArray
```

✅ This seeds the database with initial blog post data.

---

## 🔐 7. Environment Variables

### Backend `.env.sample`

```env
MONGODB_URI="mongodb://mongo/wanderlust"
#REDIS_URL="127.0.0.1:6379"
CORS_ORIGIN="http://address-of-your-ec2-instance:5173"
```

🧠 **Explanation**:
- `MONGODB_URI`: Connects to the MongoDB service using internal Docker networking.
- `CORS_ORIGIN`: Whitelists requests coming from the frontend server (replace with your EC2 public IP if needed).
- `REDIS_URL`: Placeholder for future Redis support.

---

### Frontend `.env.sample`

```env
VITE_API_PATH="http://address-of-your-ec2-instance:5000"
```

🧠 **Explanation**:
- Used by Vite at runtime to point API requests to the backend server.
- Ensure the EC2 instance's ports (`5173` for frontend, `5000` for backend) are **open in your AWS Security Group** settings.

---

## 🌐 Accessing the App on EC2

If your EC2 instance has a public IP (e.g., `13.127.182.14`), access:

- Frontend → `http://address-of-your-ec2-instance:5173`
- Backend API → `http://address-of-your-ec2-instance:5000`

---

## ✅ Summary

| Component | Description                            |
|-----------|----------------------------------------|
| MongoDB   | Data persistence via bind mount        |
| Backend   | Node.js API, port `5000`               |
| Frontend  | Vite Dev Server, port `5173`           |
| Compose   | Manages build/run of all services      |
| Volumes   | Enable Mongo data import/persistence   |

---

## 🧯 Common Errors & Troubleshooting

### ❌ `frontend` not accessible in browser

- **Symptoms**: Browser shows "Site can’t be reached" or connection times out.
- **Fixes**:
  - Ensure `--host` is included in the `CMD` of `frontend/Dockerfile`.
  - Confirm port `5173` is open in your EC2 **security group rules**.
  - Run `docker logs frontend` to check for runtime issues.

---

### ❌ `backend` can't connect to MongoDB

- **Symptoms**: Logs show `MongoNetworkError` or crash on startup.
- **Fixes**:
  - Ensure the Mongo URI in `.env.sample` is `mongodb://mongo/wanderlust`.
  - Check MongoDB is running: `docker ps` and `docker logs mongo`.
  - Restart all containers: `docker-compose down && docker-compose up --build`.

---

### ❌ `mongoimport` fails with "No such file or directory"

- **Symptoms**: The command says it can't find `sample_posts.json`.
- **Fixes**:
  - Make sure you're inside the container: `docker exec -it mongo bash`
  - Use correct path inside container: `/data/sample_posts.json`
  - Confirm the file exists locally under `backend/data` and is mounted properly.

---

### ❌ `.env.sample` not applied

- **Symptoms**: Backend or frontend doesn't reflect env variable values.
- **Fixes**:
  - Confirm `.env.sample` is copied as `.env.local` in the Dockerfile.
  - Ensure `env_file:` is set in `docker-compose.yml` for both services.
  - Rebuild containers if changes were made: `docker-compose up --build`.

---

### ❌ API CORS Error in browser

- **Symptoms**: Console shows: *“Blocked by CORS policy”* when frontend calls API.
- **Fixes**:
  - Update `CORS_ORIGIN` in `backend/.env.sample` to match your frontend URL, e.g.:  
    `http://<your-ec2-ip>:5173`
  - Rebuild and restart backend container.

---

### ❌ "Connection refused" when accessing app via EC2 public IP

- **Fixes**:
  - Open required ports (`5173`, `5000`, `27017`) in AWS EC2 **Security Group**.
  - Use EC2's **public IP** instead of `localhost`.
  - Add `--host` to Vite command so frontend is reachable externally.

---
