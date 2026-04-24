# Prep4 — Music Streaming API

A RESTful API backend for a music streaming platform built with **Node.js**, **Express.js**, and **MongoDB**. It features role-based authentication that allows **artists** to upload music and create albums, while **regular users** can browse and discover content.

---

## Table of Contents

- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Data Models](#data-models)
- [Environment Variables](#environment-variables)
- [Available Scripts](#available-scripts)
- [API Endpoints](#api-endpoints)
- [Authentication Flow](#authentication-flow)
- [Key Features](#key-features)
- [Getting Started](#getting-started)
- [Dependencies](#dependencies)
- [Notes & Observations](#notes--observations)

---

## Tech Stack

| Technology | Purpose |
|---|---|
| **Node.js & Express.js** | Server framework |
| **MongoDB + Mongoose** | Database & ODM |
| **JWT (jsonwebtoken)** | Authentication (stored in cookies) |
| **bcryptjs** | Password hashing |
| **Multer** | In-memory file upload handling |
| **ImageKit** | Cloud storage for music files |
| **dotenv** | Environment variable management |
| **cookie-parser** | Cookie parsing middleware |

---

## Project Structure

```
prep4/
├── .gitignore
├── package.json
├── package-lock.json
├── server.js                     # Entry point - starts server & connects DB
└── src/
    ├── app.js                    # Express app setup & route mounting
    ├── controllers/
    │   ├── auth.controller.js    # Register, login, logout logic
    │   └── music.controller.js   # Music upload, album CRUD, fetch logic
    ├── db/
    │   └── db.js                 # MongoDB connection handler
    ├── middlewares/
    │   └── auth.middleware.js    # JWT verification & role-based access control
    ├── models/
    │   ├── album.model.js        # Album schema
    │   ├── music.model.js        # Music / Track schema
    │   └── user.model.js         # User schema
    ├── routes/
    │   ├── auth.routes.js        # Auth endpoints
    │   └── music.routes.js       # Music & album endpoints
    └── services/
        └── storage.service.js    # ImageKit upload integration
```

---

## Data Models

### User

| Field | Type | Constraints |
|---|---|---|
| `username` | `String` | required, unique |
| `email` | `String` | required, unique |
| `password` | `String` | required |
| `role` | `String` | enum: `['user', 'artist']`, default: `'user'` |

```javascript
{
  username: "john_doe",
  email: "john@example.com",
  password: "$2a$10$...",  // hashed
  role: "artist"
}
```

### Music

| Field | Type | Constraints |
|---|---|---|
| `uri` | `String` | required — ImageKit file URL |
| `title` | `String` | required |
| `artist` | `ObjectId` | required, ref: `user` |

```javascript
{
  uri: "https://ik.imagekit.io/.../music_1712345678901",
  title: "Summer Vibes",
  artist: "507f1f77bcf86cd799439011"
}
```

### Album

| Field | Type | Constraints |
|---|---|---|
| `title` | `String` | required |
| `musics` | `[ObjectId]` | ref: `music` |
| `artist` | `ObjectId` | required, ref: `user` |

```javascript
{
  title: "My Greatest Hits",
  musics: ["507f1f77bcf86cd799439012", "507f1f77bcf86cd799439013"],
  artist: "507f1f77bcf86cd799439011"
}
```

---

## Environment Variables

Create a `.env` file in the root directory with the following variables:

```env
MONGO_URI=your_mongodb_connection_string
JWT_SECRET=your_jwt_secret_key
IMAGEKIT_PRIVATE_KEY=your_imagekit_private_key
```

## Available Scripts

| Script | Command | Description |
|---|---|---|
| **Start** | `npm start` | Runs the server in production mode with Node.js |
| **Dev** | `npm run dev` | Runs the server in development mode with Nodemon (auto-reload on file changes) |
| **Test** | `npm test` | Placeholder — no tests configured yet |

---

## API Endpoints

### Base URL

```
http://localhost:3000/api
```

### Authentication (`/api/auth`)

| Method | Endpoint | Description | Auth Required |
|---|---|---|---|
| `POST` | `/api/auth/register` | Register a new user. Optionally pass `role: "artist"`. | No |
| `POST` | `/api/auth/login` | Login with `username` or `email` and `password`. | No |
| `POST` | `/api/auth/logout` | Clear the authentication cookie. | No |

**Register / Login Request Body:**

```json
{
  "username": "john_doe",
  "email": "john@example.com",
  "password": "securepassword",
  "role": "artist"
}
```

**Success Response:**
- Sets an HTTP-only cookie named `token` containing the JWT.
- Returns user details (excluding password).

---

### Music & Albums (`/api/music`)

| Method | Endpoint | Description | Access Level |
|---|---|---|---|
| `POST` | `/api/music/upload` | Upload a music file. Accepts `multipart/form-data` with field `music`. | **Artist only** |
| `POST` | `/api/music/album` | Create a new album and assign existing music tracks to it. | **Artist only** |
| `GET` | `/api/music/` | Retrieve all music tracks (limited to 10). Populates artist info (`username`, `email`). | Authenticated users |
| `GET` | `/api/music/albums` | Retrieve all albums with artist info (`username`, `email`). | Authenticated users |
| `GET` | `/api/music/albums/:albumId` | Retrieve a single album by ID with full artist and track details. | Authenticated users |

**Create Album Request Body:**

```json
{
  "title": "My Greatest Hits",
  "musics": ["65a1b2c3d4e5f6g7h8i9j0k1", "65a1b2c3d4e5f6g7h8i9j0k2"]
}
```

**Upload Music Request (multipart/form-data):**

```
POST /api/music/upload
Content-Type: multipart/form-data

- title: "Summer Vibes"
- music: <binary file>
```

---

## Authentication Flow

1. **Registration / Login**
   - Client sends credentials to `/api/auth/register` or `/api/auth/login`.
   - Server validates input, hashes the password (bcrypt), and generates a JWT.
   - JWT is stored in an HTTP-only cookie named `token`.

2. **Protected Routes**
   - On subsequent requests, the client sends the `token` cookie automatically.
   - `auth.middleware.js` verifies the JWT using `process.env.JWT_SECRET`.
   - Decoded payload (`id`, `role`) is attached to `req.user` for downstream use.

3. **Role-Based Access Control**
   - `authUser` — grants access to both `user` and `artist` roles.
   - `authArtist` — restricts access strictly to users with the `artist` role.

---

## Key Features

- **Role-Based Access Control (RBAC)**  
  Distinct `user` and `artist` roles with middleware-enforced permissions.

- **Secure Authentication**  
  Passwords hashed with **bcryptjs** (salt rounds: 10). Sessions managed via **JWT** stored in cookies.

- **Cloud File Storage**  
  Music files are uploaded to **ImageKit** using base64 encoding via the `@imagekit/nodejs` SDK.

- **Relational Data Population**  
  Mongoose `populate()` is used to fetch associated artist details and track listings in album queries.

- **Modular Architecture**  
  Clean separation across controllers, services, middlewares, models, and routes for maintainability and scalability.

---

## Getting Started

### Prerequisites

- [Node.js](https://nodejs.org/) installed
- [MongoDB](https://www.mongodb.com/) instance (local or Atlas)
- [ImageKit](https://imagekit.io/) account (for file storage)

### Installation

1. **Clone the repository**

   ```bash
   git clone <repo-url>
   cd prep4
   ```

2. **Install dependencies**

   ```bash
   npm install
   ```

3. **Set up environment variables**

   Create a `.env` file in the project root and add your credentials:

   ```env
   MONGO_URI=mongodb+srv://<user>:<password>@cluster.mongodb.net/<dbname>
   JWT_SECRET=your_super_secret_key
   IMAGEKIT_PRIVATE_KEY=your_imagekit_private_key
   ```

4. **Run the server**

   ```bash
   # Development mode (with auto-reload)
   npm run dev

   # Production mode
   npm start
   ```

5. **Access the API**

   The server will be running at:

   ```
   http://localhost:3000
   ```

---

## Dependencies

| Package | Version | Purpose |
|---|---|---|
| `@imagekit/nodejs` | ^7.3.0 | ImageKit SDK for cloud file uploads |
| `bcryptjs` | ^3.0.3 | Password hashing |
| `cookie-parser` | ^1.4.7 | Parse and handle cookies |
| `dotenv` | ^17.4.0 | Load environment variables from `.env` |
| `express` | ^5.2.1 | Web framework |
| `jsonwebtoken` | ^9.0.3 | JWT generation and verification |
| `mongoose` | ^9.4.1 | MongoDB object modeling |
| `multer` | ^2.1.1 | Handle multipart/form-data (file uploads) |

---

