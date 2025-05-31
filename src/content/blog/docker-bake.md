---
author: Abhijeet Yadav
pubDatetime: 2025-05-31T22:44:00+05:45
modDatetime: 2025-05-31T22:44:00+05:45

title: Why Docker Bake is the Smarter Way to Build Images?
featured: false
draft: false
tags:
  - docker bake
  - bake
  - image
  - cloud
canonicalURL: https://smale.codes/posts/setting-dates-via-git-hooks/
description: Why Docker Bake is the Smarter Way to Build Images?
---

## Table of Contents

- What Docker Bake is and why it’s useful  
- How to write a `docker-bake.yaml` file using YAML instead of HCL  
- Advanced features like multiple targets and matrix builds  
- Using Docker Bake in a simple CI/CD pipeline  

---

# Docker Compose vs Docker Bake

Let’s clarify the difference:

**Docker Compose** is all about running services — managing containers, networks, volumes, and environment variables. It’s ideal for runtime orchestration.

```yaml
version: '3.8'

services:
  web:
    image: node:18
    container_name: web_app
    working_dir: /app
    volumes:
      - ./app:/app
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://user:password@db:5432/mydatabase
    depends_on:
      - db
    command: sh -c "npm install && npm start"

  db:
    image: postgres:15
    container_name: postgres_db
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
      POSTGRES_DB: mydatabase
    volumes:
      - db_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

volumes:
  db_data:
```

**Docker Bake**, on the other hand, is focused on building container images — managing build arguments, tags, and build contexts. It’s optimized for creating standardized, efficient builds.

```yaml
group:
  default:
    targets:
      - app
      - web

target:
  app:
    context: ./app
    dockerfile: Dockerfile
    tags:
      - myorg/app:latest
    platforms:
      - linux/amd64
      - linux/arm64

  web:
    context: ./web
    dockerfile: Dockerfile
    tags:
      - myorg/web:latest
    platforms:
      - linux/amd64
```

In most real-world projects, you’ll end up using both tools together — Compose for local development, and Bake for standardized builds.

---

## 🧪 Basic Usage: A Simple Flask App

Let's start with a simple Flask application:

1. A `Dockerfile` using Python 3.9  
2. A `requirements.txt` with Flask  
3. A basic Flask app exposing port 5000  

### 1. `Dockerfile`

```dockerfile
# Use the official Python 3.9 base image
FROM python:3.9-slim

# Set working directory
WORKDIR /app

# Copy requirements and install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY app.py .

# Expose port Flask will run on
EXPOSE 5000

# Run the Flask app
CMD ["python", "app.py"]
```

### 2. `requirements.txt`

```txt
flask
```

### 3. `app.py`

```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def home():
    return "Hello from Flask running in Docker!"

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
```

Previously, we would run:

```bash
docker build -t myapp:latest .
```

But with Docker Bake, we define a `docker-bake.yaml` like this:

```yaml
services:
  myapp:
    build:
      context: .
      dockerfile: Dockerfile
    image: myapp:latest
```

Then, we build it using:

```bash
docker buildx bake -f docker-bake.yaml
```

That’s it! The image is built with the tag `myapp:latest`.

---

## ⚙️ Advanced Usage: Multi-Service Builds

Docker Bake really shines in complex scenarios.

Imagine a project with:

- A frontend (React)  
- A backend (Flask)  
- A database (Postgres)  

Each component has its own Dockerfile in separate directories. Instead of three separate build commands, we define all three in a single `docker-bake.yaml` file with different targets:

```yaml
target:
  frontend:
    context: ./frontend
    dockerfile: Dockerfile
    tags: ["frontend:latest"]

  backend:
    context: ./backend
    dockerfile: Dockerfile
    tags: ["backend:latest"]

  database:
    context: ./database
    dockerfile: Dockerfile
    tags: ["database:latest"]
```

Then, with one command, we can build them all:

```bash
docker buildx bake -f docker-bake.yaml
```

This simplifies your workflow drastically and ensures consistency across all your services.

---

## 🌍 Environment-Specific Builds

You can even define multiple environments (like dev, QA, and production) using conditional logic within the bake file. For example:

```yaml
target:
  dev:
    context: .
    args:
      ENV: development
    tags: ["myapp:dev"]

  prod:
    context: .
    args:
      ENV: production
    tags: ["myapp:prod"]
```

The same Dockerfile can behave differently based on the passed `ENV` argument, making it easy to maintain multiple configurations in one place.

---

## 🔁 CI/CD Integration

Docker Bake works great in CI/CD pipelines. In fact, you can define matrix builds for multiple platforms and targets like this:

```yaml
group:
  default:
    targets: ["api-dev", "api-prod", "web-dev", "web-prod"]

target:
  base:
    platforms: ["linux/amd64", "linux/arm64"]

  api-dev:
    inherits: ["base"]
    context: ./api
    args:
      ENV: development

  api-prod:
    inherits: ["base"]
    context: ./api
    args:
      ENV: production

  web-dev:
    inherits: ["base"]
    context: ./web
    args:
      ENV: development

  web-prod:
    inherits: ["base"]
    context: ./web
    args:
      ENV: production
```

Run it with:

```bash
docker buildx bake -f docker-bake.yaml
```

This structure makes it ideal for GitHub Actions, GitLab CI, or any other CI tool that benefits from standardized builds across different environments and platforms.

---

## ✅ Recap: Why Use Docker Bake?

| Docker Bake                            | Docker Compose                      |
|----------------------------------------|-------------------------------------|
| Builds container images                | Runs containerized services         |
| Handles arguments, tags, platforms     | Manages volumes, networks, env vars |
| Great for CI/CD                        | Ideal for local development         |
| Excels at matrix builds                | Excels at service orchestration     |

Use **Docker Bake** for building.  
Use **Docker Compose** for running.  
Together, they make a powerful DevOps duo.

---

## 🏁 Final Thoughts

Getting started is simple — just install Docker Desktop 4.38, create a `docker-bake.yaml`, and start building smarter!

**Until next time,**  
**Happy baking with Docker!**