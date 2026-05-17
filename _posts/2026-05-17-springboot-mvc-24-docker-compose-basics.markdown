---
layout: post
title: "SpringBoot Intro 24: Docker Compose Basics"
date: 2026-05-17 22:02:10 +0900
category: SpringBoot
permalink: /springboot/mvc-docker-compose-basics
---

# SpringBoot Intro 24: Docker Compose Basics

In the previous post, we ran one Spring Boot container with a simple Dockerfile.
Now we connect app + database together using Docker Compose.

Goal:

- Start Spring Boot and MySQL together
- Keep setup reproducible for local development

## 1) Minimal `docker-compose.yml`

```yaml
services:
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: demo
      MYSQL_USER: demo
      MYSQL_PASSWORD: demo
    ports:
      - "3306:3306"

  app:
    build: .
    depends_on:
      - db
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://db:3306/demo
      SPRING_DATASOURCE_USERNAME: demo
      SPRING_DATASOURCE_PASSWORD: demo
    ports:
      - "8080:8080"
```

## 2) Run

```bash
docker compose up --build
```

Then open `http://localhost:8080`.

## 3) Why this helps beginners

- One command starts all required services
- Team members can share the same dev environment
- Easy reset with `docker compose down -v`

## Summary

- Docker runs one container well
- Docker Compose manages multiple containers as one app stack
- For Spring Boot + DB projects, Compose is a practical next step
