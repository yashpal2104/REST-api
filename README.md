# REST-api

Simple Go REST API project (tasks, users, store abstraction).

## Overview

This repository contains a minimal REST API written in Go. It defines a `Store` interface and a `Storage` implementation that uses a SQL database. The `TasksService` registers HTTP handlers that use the `Store` to create and fetch tasks.

## Prerequisites

- Go 1.20+ installed
- A SQL database (the code expects a *sql.DB injected into `NewStore`)
- `github.com/gorilla/mux` (go get will fetch this automatically when building)

## Build & Run

1. Fetch modules and build:

```bash
go mod tidy
go build ./...
```

2. Run the server (example):

```bash
# set up your DB and provide connection in main.go, then:
go run main.go
```

## Endpoints

- POST /tasks — create a task (JSON payload)
- GET /tasks/{id} — fetch a task by ID

Example JSON payload for creating a task:

```json
{
  "name": "Implement feature X",
  "project_id": 1,
  "assigned_to_id": 2
}
```

Server responsibilities for `handleCreateTask`:
- Read and validate the request payload
- Call `s.store.CreateTask(task)` to persist
- Return `201 Created` with the created task on success, or a 4xx/5xx error payload on failure

## Notes

- `TasksService` stores `store Store` as an interface value (idiomatic). Concrete implementations like `*Storage` with pointer receivers satisfy the interface.
- Handlers use pointer receivers (`*TasksService`) to avoid copying, to allow mutation, and to safely hold synchronization primitives in the future.

## Contributing

Open a PR or submit issues.

---
Created automatically.
