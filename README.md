# c2s-webscraping-rails

Ruby on Rails microservices ecosystem for managing vehicle-ad listing web scraping tasks, with JWT authentication, asynchronous processing through Sidekiq, and lifecycle notification tracking.

![](images/c2s-webscraping-rails-infographic.webp)

## 🏗️ Arquitetura

```mermaid
flowchart TD
	subgraph ClientSide["👤 Client"]
		CLI(["🖥️ Browser (Web UI)"])
	end

	subgraph CoreServices["⚙️ Rails Microservices"]
		AUTH["🔐 auth-service"]
		MANAGER["🧭 webscraping-manager"]
		PROC["🕷️ processing-service"]
		NOTIF["🔔 notification-service"]
		SIDEKIQ["⚙️ webscraping-manager-sidekiq"]
	end

	subgraph Infra["🗄️ Infrastructure"]
		PG[("🗄️ PostgreSQL")]
		REDIS[("🧠 Redis")]
	end

	subgraph External["🌐 External Website"]
		WEB["🌐 Webmotors"]
	end

	CLI -->|"web pages + actions"| MANAGER
	MANAGER -->|"registration/login"| AUTH
	MANAGER -->|"CRUD task"| PG
	MANAGER -->|"enqueue job"| REDIS
	REDIS -->|"consume queue"| SIDEKIQ
	SIDEKIQ -->|"scrape task"| PROC
	PROC -->|"scrape listing"| WEB
	PROC -->|"return brand/model/price"| SIDEKIQ
	SIDEKIQ -->|"update task"| MANAGER
	SIDEKIQ -->|"publish event"| NOTIF
	AUTH -->|"users"| PG
	NOTIF -->|"notifications"| PG
```

## 🔁 Sequence Diagrams

<details>
	<summary><strong>auth-service (registration and login)</strong></summary>

```mermaid
sequenceDiagram
	autonumber
	participant C as 👤 Client
	participant A as 🔐 auth-service
	participant DB as 🗄️ PostgreSQL

	C->>A: POST /api/v1/auth/register
	A->>DB: INSERT user
	DB-->>A: user created
	A-->>C: 201 user + token + exp

	C->>A: POST /api/v1/auth/login
	A->>DB: SELECT user by email
	DB-->>A: user
	A-->>C: 200 user + token + exp

	C->>A: GET /health
	A-->>C: 200 {status: ok}
```

</details>

<details>
	<summary><strong>webscraping-manager (task API)</strong></summary>

```mermaid
sequenceDiagram
	autonumber
	participant C as 👤 Client
	participant M as 🧭 webscraping-manager
	participant DB as 🗄️ PostgreSQL
	participant N as 🔔 notification-service
	participant R as 🧠 Redis/Sidekiq

	C->>M: POST /api/v1/tasks (Bearer, title, ad_url)
	M->>DB: INSERT task (status=pending)
	M->>N: POST /api/v1/notifications (task_created)
	M->>R: enqueue TaskProcessingWorker(task_id)
	M-->>C: 201 task

	C->>M: GET /api/v1/tasks
	M->>DB: SELECT tasks ORDER BY created_at DESC
	M-->>C: 200 list

	C->>M: GET /api/v1/tasks/:id
	M->>DB: SELECT task by id
	M-->>C: 200 task

	C->>M: DELETE /api/v1/tasks/:id
	M->>DB: DELETE task
	M-->>C: 204 no_content
```

</details>

<details>
	<summary><strong>webscraping-manager-sidekiq (asynchronous worker)</strong></summary>

```mermaid
sequenceDiagram
	autonumber
	participant Q as 🧠 Sidekiq Queue
	participant W as ⚙️ TaskProcessingWorker
	participant DB as 🗄️ PostgreSQL
	participant P as 🕷️ processing-service
	participant N as 🔔 notification-service

	Q->>W: perform(task_id)
	W->>DB: find task
	alt task does not exist or is terminal
		W-->>Q: return
	else task is pending
		W->>DB: update status=processing
		W->>P: POST /api/v1/scrape (task_id, ad_url)
		alt status == completed
			P-->>W: {status, brand, model, price}
			W->>DB: update status=completed + data + completed_at
			W->>N: POST notification (task_completed)
		else error/failure
			P-->>W: {status=failed, error_message}
			alt anti-bot/captcha detected and retries remain
				W->>DB: update status=pending + retry message
				W-->>Q: perform_in(backoff, task_id, retry_count+1)
			else final failure
				W->>DB: update status=failed + error_message + completed_at
				W->>N: POST notification (task_failed)
			end
		end
	end
```

</details>

<details>
	<summary><strong>processing-service (scrape)</strong></summary>

```mermaid
sequenceDiagram
	autonumber
	participant W as ⚙️ Worker (manager)
	participant P as 🕷️ processing-service
	participant S as 🌐 Target Website (Webmotors)

	W->>P: POST /api/v1/scrape (task_id, ad_url)
	P->>S: GET ad_url
	S-->>P: HTML
	P->>P: parse Nokogiri
	alt success
		P-->>W: 200 {status: completed, brand, model, price, error_message: null}
	else anti-bot/captcha block
		P-->>W: 200 {status: failed, error_message: "anti-bot block"}
	else failure
		P-->>W: 200 {status: failed, error_message}
	end

	W->>P: GET /health
	P-->>W: 200 {status: ok}
```

</details>

<details>
	<summary><strong>notification-service (events)</strong></summary>

```mermaid
sequenceDiagram
	autonumber
	participant M as 🧭 webscraping-manager/worker
	participant N as 🔔 notification-service
	participant DB as 🗄️ PostgreSQL
	participant C as 👤 Client

	M->>N: POST /api/v1/notifications
	N->>DB: INSERT notification
	DB-->>N: persisted
	N-->>M: 201 notification

	C->>N: GET /api/v1/notifications
	N->>DB: SELECT notifications ORDER BY created_at DESC
	N-->>C: 200 list

	C->>N: GET /health
	N-->>C: 200 {status: ok}
```

</details>

## 🧩 Services

- 🧭 `webscraping-manager`: Web UI (login/registration + tasks) + task API (`create`, `index`, `show`, `destroy`) + reprocessing action.
- ⚙️ `webscraping-manager-sidekiq`: dedicated worker for asynchronous task processing.
- 🔐 `auth-service`: registration/login and JWT issuance with expiration.
- 🕷️ `processing-service`: scraping with Nokogiri/HTTP and a standardized response (`completed`/`failed`).
- 🔔 `notification-service`: event persistence and listing (`task_created`, `task_completed`, `task_failed`).
- 🗄️ Shared infrastructure: `postgres` + `redis`.

## 🧱 Tech Stack

- 💎 Ruby on Rails
- 🗄️ PostgreSQL
- 🧠 Redis + ⚙️ Sidekiq
- 🔐 JWT
- 🕸️ Nokogiri
- 🌐 HTTParty
- 🐳 Docker Compose
- 🧪 RSpec + 🧹 Rubocop

## 🗂️ Project Structure

```text
.
├── auth-service/
├── notification-service/
├── processing-service/
├── webscraping-manager/
├── docker-compose.yml
└── README.md
```

### 📥 Clone Repositories (Main Project + Services)

> 📌 Important: service repositories must be cloned **inside** the `c2s-webscraping-rails` repository (as sibling subdirectories), as shown above.

```bash
# 1) Clone the main repository
git clone https://github.com/enogrob/c2s-webscraping-rails.git
cd c2s-webscraping-rails

# 2) Clone the services (subdirectories)
git clone https://github.com/enogrob/webscraping-manager.git
git clone https://github.com/enogrob/processing-service.git
git clone https://github.com/enogrob/notification-service.git
git clone https://github.com/enogrob/auth-service.git
```

## ✅ Prerequisites

- 🐳 Docker
- 🧩 Docker Compose

## ⚙️ Environment Variables

Each service includes version-controlled templates to simplify setup:

- `.env.example` (reference/compose)
- `.env.test.example` (reference for local tests)

The actual `.env` and `.env.test` files **must not be committed** (they are ignored by Git). For local use, copy the templates into each service:

```bash
cp .env.example .env
cp .env.test.example .env.test
```

`DATABASE_URL` notes:

- Rails running on the host + Postgres via `docker compose` (port `55432`): use `postgresql://postgres:postgres@localhost:55432/...`
- Rails running inside a container: use `postgresql://postgres:postgres@postgres:5432/...` (where `postgres` is the service name in Compose)
- Postgres running locally (installed on the host, default port `5432`): use `postgresql://localhost/...` or `postgresql://USER:PASSWORD@localhost:5432/...`

- 🔐 `auth-service/.env.example`
- 🔔 `notification-service/.env.example`
- 🕷️ `processing-service/.env.example`
- 🧭 `webscraping-manager/.env.example`

- 🔐 `auth-service/.env.test.example`
- 🔔 `notification-service/.env.test.example`
- 🕷️ `processing-service/.env.test.example`
- 🧭 `webscraping-manager/.env.test.example`

## ▶️ How to Run (One Command)

From the root of this repository (`src/c2s-webscraping-rails`):

```bash
docker compose up -d
```

Compose starts:

- 🧭 `webscraping-manager` (host `3000`)
- 🔐 `auth-service` (host `3001`)
- 🔔 `notification-service` (host `3002`)
- 🕷️ `processing-service` (host `3003`)
- ⚙️ `webscraping-manager-sidekiq`
- 🗄️ `postgres` (host `55432`, container `5432`)
- 🧠 `redis` (host `6379`)

## 🗄️ Prepare the Database

After starting the containers, run:

```bash
docker compose exec auth-service bundle exec rails db:prepare
docker compose exec notification-service bundle exec rails db:prepare
docker compose exec webscraping-manager bundle exec rails db:prepare
```

## 🩺 Health Checks

```bash
curl http://localhost:3000/health
curl http://localhost:3001/health
curl http://localhost:3002/health
curl http://localhost:3003/health
```

Expected response:

```json
{"status":"ok"}
```

## 🔌 Main Endpoints (MVP)

### auth-service

- `🟠 POST /api/v1/auth/register`
- `🟠 POST /api/v1/auth/login`
- `🟢 GET /health`

### webscraping-manager (API)

- `🟠 POST /api/v1/tasks`
- `🟢 GET /api/v1/tasks`
- `🟢 GET /api/v1/tasks/:id`
- `🔴 DELETE /api/v1/tasks/:id`
- `🟢 GET /health`

### webscraping-manager (Web UI)

- `🟢 GET /login`
- `🟠 POST /login`
- `🟢 GET /register`
- `🟠 POST /register`
- `🟢 GET /tasks`
- `🟢 GET /tasks/:id`
- `🟠 POST /tasks/:id/reprocess`
- `🔴 DELETE /tasks/:id`
- `🔴 DELETE /logout`

<table>
	<tr>
		<td align="center">
			<a href="images/screenshot_220.png" target="_blank" rel="noopener noreferrer">
				<img src="images/screenshot_220.png" alt="Login" width="260" />
			</a>
			<br />
			<sub>Login</sub>
		</td>
		<td align="center">
			<a href="images/screenshot_221.png" target="_blank" rel="noopener noreferrer">
				<img src="images/screenshot_221.png" alt="Tasks" width="260" />
			</a>
			<br />
			<sub>List</sub>
		</td>
		<td align="center">
			<a href="images/screenshot_222.png" target="_blank" rel="noopener noreferrer">
				<img src="images/screenshot_222.png" alt="Details" width="260" />
			</a>
			<br />
			<sub>Details</sub>
		</td>
	</tr>
</table>

### Error Pages (Web UI)


- `🟢 GET /400.html`
- `🟢 GET /401.html`
- `🟢 GET /404.html`
- `🟢 GET /422.html`
- `🟢 GET /500.html`

### processing-service

- `🟠 POST /api/v1/scrape`
- `🟢 GET /health`

### notification-service

- `🟠 POST /api/v1/notifications`
- `🟢 GET /api/v1/notifications`
- `🟢 GET /health`

## 🔄 Functional Flow Summary

1. 👤 The user registers/logs in to `webscraping-manager` (through `auth-service`).
2. 📝 The user creates a scraping task.
3. 🧭 `webscraping-manager` creates a `pending` task and enqueues a job.
4. ⚙️ `webscraping-manager-sidekiq` calls `processing-service`.
5. ✅ The task becomes `completed` (with `brand/model/price`) or `failed` (with `error_message`).
6. 🧱 If an anti-bot/captcha block occurs, the worker retries with backoff (up to 3 attempts) before the final failure.
7. 🔔 The `task_failed` event is published only when the failure is final.

## 🗺️ Visual Maps (Quick Overview)

<details>
	<summary><strong>🔁 Task Lifecycle (status + retries)</strong></summary>

```mermaid
stateDiagram-v2
	state "🟡 pending" as pending
	state "🔵 processing" as processing
	state "✅ completed" as completed
	state "🔴 failed" as failed

	[*] --> pending: 📨 enqueue job
	pending --> processing: 🧠 dequeue (Sidekiq)

	processing --> completed: 🕷️ scrape succeeds
	completed --> [*]: 🔔 task_completed

	processing --> failed: 🕷️ scrape fails
	failed --> pending: ⏱️ retry/backoff (up to 3)\n🧱 anti-bot/captcha
	failed --> [*]: 🔔 task_failed (final)

	completed --> pending: 🔁 reprocess (Web UI)
	failed --> pending: 🔁 reprocess (Web UI)
```

</details>

<details>
	<summary><strong>🗄️ Data Model (Simplified)</strong></summary>

```mermaid
erDiagram
	direction LR
	USER ||--o{ TASK : "🧭 creates 📝"
	TASK ||--o{ NOTIFICATION : "🔔 generates 📣"

	USER {
		int id PK
		string email
	}

	TASK {
		int id PK
		string status
		datetime completed_at
	}

	NOTIFICATION {
		int id PK
		string event_type
		int task_id FK
		datetime created_at
	}
```

In the ER diagram, entity names remain without emoji because they are also schema identifiers, and Mermaid does not provide separate display labels.

</details>

## 🧪 Tests

Run the tests for each service:

```bash
cd auth-service && bundle exec rspec && cd .. 
cd notification-service && bundle exec rspec && cd ..
cd processing-service && bundle exec rspec && cd ..
cd webscraping-manager && bundle exec rspec && cd ..
```

Example focusing on the manager's asynchronous flow:

```bash
cd webscraping-manager
bundle exec rspec spec/requests/api/v1/task_lifecycle_spec.rb spec/requests/api/v1/tasks_spec.rb spec/workers/task_processing_worker_spec.rb
```

## 🧹 Lint

```bash
cd auth-service && bundle exec rubocop && cd ..
cd notification-service && bundle exec rubocop && cd ..
cd processing-service && bundle exec rubocop && cd ..
cd webscraping-manager && bundle exec rubocop && cd ..
```

## 🔗 Referências

* [c2s-webscraping-rails](https://github.com/enogrob/c2s-webscraping-rails)
* [webscraping-manager](https://github.com/enogrob/webscraping-manager)
* [processing-service](https://github.com/enogrob/processing-service)
* [notification-service](https://github.com/enogrob/notification-service)
* [auth-service](https://github.com/enogrob/auth-service)

