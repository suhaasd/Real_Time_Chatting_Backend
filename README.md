# PingTalk — Real-Time Chat Backend

A Node.js microservices backend for a real-time chatting application. Four independent Express services communicate via RabbitMQ events and are exposed through an NGINX API gateway. MongoDB is used for persistence, Redis for caching, Socket.IO for real-time messaging, and Cloudinary for file storage.

---

## Architecture Overview

```
Client
  │
  ▼
NGINX API Gateway  (port 3000)
  ├── /auth/*       → Auth Service      (port 5000)
  ├── /profile/*    → Profile Service   (port 5001)
  ├── /request/*    → Request Service   (port 5002)
  └── /chat/*       → Chat Service      (port 5003)
```

Services communicate asynchronously via **RabbitMQ** events:

```
Auth Service       ──auth.user.created──►  Profile Service
                                        ►  Request Service

Request Service    ──request.accepted──►   Chat Service (creates Connection)
                   ──request.rejected──►
```

---

## Services

### Auth Service — Port 5000

Handles user registration, login, logout, and JWT issuance.

| Method | Endpoint | Description |
|---|---|---|
| POST | `/signup` | Register a new user |
| POST | `/login` | Authenticate and set JWT cookie |
| POST | `/logout` | Clear auth cookie |
| PATCH | `/changePassword` | Update password (authenticated) |
| GET | `/getUserByEmail` | Look up a user by email |

- Passwords hashed with **bcrypt**
- JWT stored as an HTTP-only cookie (24h expiry)
- Login rate-limited to 15 attempts per 15 minutes
- Publishes `auth.user.created` event on signup

---

### Profile Service — Port 5001

Manages user profile data and profile pictures.

| Method | Endpoint | Description |
|---|---|---|
| GET | `/getProfile` | Get the authenticated user's profile |
| PUT | `/patchProfile` | Update profile (with optional image upload) |
| GET | `/profile/:userId` | Get any user's profile by ID |

- Consumes `auth.user.created` to initialize profiles
- Profile photos uploaded to **Cloudinary**
- Profiles cached in **Redis** (10-minute TTL)

---

### Request Service — Port 5002

Manages friend/connection invites between users.

| Method | Endpoint | Description |
|---|---|---|
| POST | `/invite/send` | Send a connection invite by email |
| PATCH | `/respond/:requestId` | Accept or reject a received request |
| GET | `/invites/sent` | List sent requests (with status) |
| GET | `/invites/received` | List received pending requests |
| DELETE | `/cancel/:requestId` | Cancel a pending sent request |

- Request lists cached in **Redis** (10-minute TTL)
- Publishes `request.accepted` / `request.rejected` events
- Consumes `auth.user.created` to keep a local user directory

---

### Chat Service — Port 5003

Handles chat creation, message history, file attachments, and real-time delivery via Socket.IO.

**HTTP Endpoints:**

| Method | Endpoint | Description |
|---|---|---|
| POST | `/create-direct` | Start a direct chat (requires connection) |
| POST | `/create-group` | Create a group chat (min 2 participants) |
| PUT | `/add-to-group` | Add a user to a group (admin only) |
| PUT | `/remove-from-group` | Remove a user from a group (admin only) |
| PUT | `/leave-group` | Leave a group |
| GET | `/my-chats` | List all chats for the authenticated user |
| GET | `/messages/:chatId` | Fetch message history for a chat |
| POST | `/send-file` | Send a file attachment (multipart, max 5 MB) |

**Socket.IO Events:**

| Event | Direction | Description |
|---|---|---|
| `joinChat` | Client → Server | Join a chat room |
| `sendMessage` | Client → Server | Send a message |
| `receiveMessage` | Server → Client | Deliver a message to participants |
| `disconnect` | Client → Server | User leaves |

- Consumes `request.accepted` to create a Connection record enabling chat access
- File attachments stored on **Cloudinary**
- Chat permissions cached in **Redis** (1-hour TTL)

---

### API Gateway (NGINX) — Port 3000

Routes all external traffic to the appropriate service.

- Single entry point for clients
- WebSocket upgrade passthrough for Socket.IO (`/chat/*`)
- Rate limiting: 10 req/sec per IP, burst 20
- CORS origin validation
- Security headers: `X-Frame-Options`, `X-Content-Type-Options`, `X-XSS-Protection`
- `/health` endpoint returns 200

---

## Tech Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js 18 |
| Framework | Express 5 |
| Database | MongoDB (Mongoose) |
| Cache | Redis |
| Message Queue | RabbitMQ |
| Real-time | Socket.IO 4 |
| File Storage | Cloudinary |
| Gateway | NGINX |
| Containerization | Docker + Docker Compose |
| CI/CD | Jenkins |

---

## Getting Started

### Prerequisites

- Docker & Docker Compose
- `.env` files for each service (see [Environment Variables](#environment-variables) below)

### Run with Docker Compose

```bash
docker compose up --build
```

All five services (four apps + NGINX gateway) start on a shared `backend-network`. The gateway is accessible at `http://localhost:3000`.

### Run a Single Service Locally

```bash
cd authService   # or profileService, requestService, chatService
npm install
npm run dev      # nodemon — auto-restarts on file changes
npm start        # production
```

---

## Environment Variables

Each service reads from a `.env` file in its own directory. The variables used across services:

| Variable | Used By | Description |
|---|---|---|
| `MONGO_URI` | All services | MongoDB connection string |
| `JWT_SECRET` | All services | Shared JWT signing secret |
| `RABBITMQ_URL` | All services | RabbitMQ connection string |
| `REDIS_HOST` | All services | Redis hostname |
| `REDIS_PORT` | All services | Redis port |
| `REDIS_PASSWORD` | All services | Redis auth password |
| `CLOUDINARY_CLOUD_NAME` | Profile, Chat | Cloudinary cloud name |
| `CLOUDINARY_API_KEY` | Profile, Chat | Cloudinary API key |
| `CLOUDINARY_API_SECRET` | Profile, Chat | Cloudinary API secret |
| `AUTH_SERVICE_PORT` | Auth | Port to listen on (default 5000) |
| `PROFILE_SERVICE_PORT` | Profile | Port to listen on (default 5001) |
| `REQUEST_SERVICE_PORT` | Request | Port to listen on (default 5002) |
| `CHAT_SERVICE_PORT` | Chat | Port to listen on (default 5003) |

---

## Data Models

**Auth** — `authDB`
- `emailId`, `password` (hashed), `firstName`, `lastName`

**Profile** — `profileDB`
- `userId`, `firstName`, `lastName`, `about`, `profilePic`, `age`, `gender`

**Request** — `requestDB`
- `fromUserId`, `toUserId`, `status` (`pending` / `accepted` / `rejected`), `message`

**Chat** — `chatDB`
- `Chat`: `chatType` (`direct` / `group`), `participants[]`, `admins[]`, `lastMessage`
- `Message`: `chatId`, `senderId`, `text`, `file` (`data`, `name`, `mimeType`, `size`), `readBy[]`
- `Connection`: `user1`, `user2` (sorted), `status`

---

## Key Flows

**Signup**
1. Client POSTs to `/auth/signup`
2. Auth Service hashes password, creates user, issues JWT cookie
3. `auth.user.created` event published to RabbitMQ
4. Profile and Request services consume the event and initialize local records

**Connect with a User**
1. Client sends invite via `/request/invite/send`
2. Recipient accepts via `/request/respond/:id`
3. `request.accepted` event published
4. Chat Service consumes event and creates a Connection record

**Send a Message**
1. Client joins a chat room via `joinChat` Socket.IO event
2. Chat Service validates the Connection exists
3. Client emits `sendMessage`; message saved to MongoDB
4. Chat Service broadcasts `receiveMessage` to all room participants
