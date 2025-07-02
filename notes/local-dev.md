# Vite + EC2 (Ubuntu) Dev Server Access Guide

This guide explains how to run a Vite development server on an EC2 Ubuntu instance and access it from your local machine, based on a real-world troubleshooting session.
It also covers deploying a production-ready Dockerized Vite app using Nginx.

---

## 🧑‍💻 Scenario: Dev Mode

You have a Vite project hosted on an Ubuntu EC2 instance. You run:

```bash
npm run dev
```

And see:

```
VITE v5.4.8  ready in 236 ms

  ➜  Local:   http://localhost:5173/
  ➜  Network: use --host to expose
```

You've already opened port **5173** in your EC2 Security Group (SG), but from your local browser you get:

```
This site can’t be reached
34.228.199.224 refused to connect.
```

---

## ✅ Solution: Use `--host` Flag

By default, Vite binds to `localhost` (127.0.0.1), which means it is only accessible inside the EC2 instance.

To make it accessible from outside:

```bash
npm run dev -- --host
```

This binds the Vite dev server to `0.0.0.0`, allowing external connections.

### Why the double `--`?

This is required to forward CLI flags through `npm run` to the underlying command (Vite, in this case).

---

## 🔒 Security Group Configuration

Ensure your EC2 instance's **Security Group** has an **inbound rule** allowing TCP traffic on port **5173**, e.g.:

* **Type**: Custom TCP
* **Port Range**: 5173
* **Source**: Your IP or `0.0.0.0/0` (not recommended for production)

---

## 🌐 Accessing the App in Dev Mode

Once the server is running with `--host`, and your SG is configured:

* Visit: `http://<your-ec2-public-ip>:5173`
* Example: `http://34.228.199.224:5173`

---

## 🐳 Deploying a Dockerized Production Build

You can also deploy your Vite app as a static site using Docker and Nginx.

### Dockerfile Example

```Dockerfile
# Build stage
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Production stage
FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```

### Build the Docker Image

```bash
docker build -t vite-app .
```

### Run the Docker Container

```bash
docker run -d -p 80:80 --name vite-container vite-app
```

### Update Security Group

Ensure port **80** is open in your EC2 SG:

* **Type**: HTTP
* **Port**: 80
* **Source**: Your IP or `0.0.0.0/0`

### Access Your App

Now visit:

```
http://<your-ec2-public-ip>/
```

---

## 🧠 Networking Concepts Explained

### 127.0.0.1 (localhost)

* Only accessible **inside the same machine**.
* Default bind address for many dev servers.

### 0.0.0.0

* Binds to **all network interfaces**.
* Required to make a server accessible externally.

### 172.31.x.x

* **Private IP** assigned by AWS to EC2 inside a **VPC**.
* Only used for **internal AWS networking** (not reachable from internet).

### Public IP (e.g. 34.x.x.x)

* Assigned by AWS for **external access**.
* Use this to access the EC2 instance from your local machine.

---

## 🏁 Final Notes

* Use `--host` for external access to dev servers.
* Use Docker + Nginx for production builds.
* Always open necessary ports in your EC2 Security Group.
* Access your instance using its **public IP**, not the internal/private IP.

For production apps, consider using Nginx with custom config and HTTPS.

---

Happy coding with Vite on the cloud! 🚀
