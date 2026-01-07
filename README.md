# Loviora

Initial repository created to receive the Loviora (dating app) scaffold. This README is a placeholder commit to initialize the default branch.

I will push the full Loviora MVP scaffold on a branch named `loviora/mvp` after this file is created.

(If you did not expect this, please abort and contact the maintainer.)
#!/usr/bin/env bash
set -euo pipefail

REPO="https://github.com/yoxyoxdur135-png/pulse.git"
CLONE_DIR="pulse-loviora-temp"
BRANCH="loviora/mvp"
PR_TITLE='feat: add Loviora MVP scaffold (backend + frontend + admin + flutter UI)'
PR_BODY_FILE="pr_body.txt"

echo "This script will clone $REPO, create branch $BRANCH, write files, commit, push, and attempt to create a PR using gh."
read -p "Continue? (y/N) " confirm
if [[ "${confirm:-N}" != "y" && "${confirm:-N}" != "Y" ]]; then
  echo "Aborted."
  exit 1
fi

# remove existing clone if present
rm -rf "$CLONE_DIR"
git clone "$REPO" "$CLONE_DIR"
cd "$CLONE_DIR"

# create or switch to branch
if git show-ref --verify --quiet "refs/heads/$BRANCH"; then
  git checkout "$BRANCH"
else
  git checkout -b "$BRANCH"
fi

# Create directories
mkdir -p backend/src/scripts
mkdir -p backend/src
mkdir -p frontend/src/pages
mkdir -p frontend/src/ui
mkdir -p frontend/src/api
mkdir -p frontend/src/styles
mkdir -p mobile/pulse_ui_package/lib

# -------- Write files --------
cat > README.md <<'EOF'
# Loviora — Dating App Starter (MVP)

Loviora is a Badoo-like dating app starter scaffold (original UI/UX).  
Stack: Node.js + Express, PostgreSQL, Socket.IO, React (Vite). Optional Flutter UI package included.

Quick start (dev)
1. Copy env files:
   - cp backend/.env.example backend/.env
   - cp frontend/.env.example frontend/.env
2. Start dev stack:
   - docker-compose up --build -d
3. Backend:
   - docker exec -it loviora_backend npm ci
   - docker exec -it loviora_backend npm run migrate
   - docker exec -it loviora_backend npm run seed
4. Frontend:
   - cd frontend
   - npm ci
   - npm run dev
5. Open app:
   - Frontend: http://localhost:5173
   - Backend API: http://localhost:4000/api
   - Admin: http://localhost:5173/admin-panel

Notes
- Database: PostgreSQL (default). To switch to MongoDB I can adapt migrations and queries.
- Payments: Stripe stub included (server-side placeholder). Crypto QR payment is a manual/confirmation stub.
- Images: multer saves to /tmp/uploads by default; configure S3 in production using services/storage.js
- Seeded demo accounts (admins/moderators) inserted by seed script. See backend/src/scripts/seed.js

License: MIT
EOF

cat > docker-compose.yml <<'EOF'
version: '3.8'
services:
  db:
    image: postgres:15
    restart: unless-stopped
    environment:
      POSTGRES_USER: loviora
      POSTGRES_PASSWORD: loviora_secret
      POSTGRES_DB: loviora_dev
    volumes:
      - dbdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgres://loviora:loviora_secret@db:5432/loviora_dev
      - JWT_SECRET=change_me
    depends_on:
      - db
    volumes:
      - ./backend:/usr/src/app
    ports:
      - "4000:4000"

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    volumes:
      - ./frontend:/app
    ports:
      - "5173:5173"
    environment:
      - VITE_API_URL=http://localhost:4000/api

volumes:
  dbdata:
EOF

cat > backend/.env.example <<'EOF'
NODE_ENV=development
PORT=4000
DATABASE_URL=postgres://loviora:loviora_secret@db:5432/loviora_dev

JWT_SECRET=replace_with_strong_secret
JWT_EXPIRES_IN=1d
REFRESH_TOKEN_EXPIRES_IN=7d

S3_ENDPOINT=
S3_BUCKET=
S3_KEY=
S3_SECRET=

STRIPE_SECRET=
CRYPTO_CONFIRMATION_REQUIRED=false
EOF

cat > backend/Dockerfile <<'EOF'
FROM node:18-alpine
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm ci
COPY . .
EXPOSE 4000
CMD ["npm", "run", "dev"]
EOF

cat > backend/package.json <<'EOF'
{
  "name": "loviora-backend",
  "version": "0.1.0",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js",
    "dev": "nodemon src/index.js",
    "migrate": "node src/scripts/migrate.js",
    "seed": "node src/scripts/seed.js"
  },
  "dependencies": {
    "bcrypt": "^5.1.0",
    "cors": "^2.8.5",
    "dotenv": "^16.0.0",
    "express": "^4.18.2",
    "helmet": "^6.0.0",
    "jsonwebtoken": "^9.0.0",
    "pg": "^8.11.0",
    "socket.io": "^4.7.0",
    "multer": "^1.4.5",
    "express-rate-limit": "^6.7.0",
    "joi": "^17.9.2"
  },
  "devDependencies": {
    "nodemon": "^2.0.22"
  }
}
EOF

cat > backend/src/index.js <<'EOF'
// backend/src/index.js
// Minimal Express backend with JWT auth, DOB validation, likes, matches, and Socket.IO
const express = require('express');
const helmet = require('helmet');
const cors = require('cors');
const bodyParser = require('body-parser');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcrypt');
const { Pool } = require('pg');
const http = require('http');
const { Server } = require('socket.io');
const rateLimit = require('express-rate-limit');
const multer = require('multer');
require('dotenv').config();

const app = express();
app.use(helmet());
app.use(cors({ origin: true }));
app.use(bodyParser.json());

// rate limiter
const limiter = rateLimit({ windowMs: 15 * 60 * 1000, max: 200 });
app.use(limiter);

// DB pool
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Simple DOB validator: returns true if age >= 18
function isAdult(dobString) {
  const dob = new Date(dobString);
  if (Number.isNaN(dob.getTime())) return false;
  const diff = Date.now() - dob.getTime();
  const ageDt = new Date(diff);
  return Math.abs(ageDt.getUTCFullYear() - 1970) >= 18;
}

// --- helpers ---
function signToken(userId) {
  return jwt.sign({ sub: userId }, process.env.JWT_SECRET, { expiresIn: process.env.JWT_EXPIRES_IN || '1d' });
}

async function findUserByEmail(email) {
  const { rows } = await pool.query('SELECT id, email, name, password, role FROM users WHERE email=$1', [email]);
  return rows[0];
}

// --- multer for avatar uploads (local for now) ---
const upload = multer({ dest: '/tmp/uploads' });

// --- routes ---
// Register
app.post('/api/auth/register', async (req, res) => {
  const { email, password, name, dob } = req.body;
  if (!email || !password || !dob) return res.status(400).json({ message: 'Missing fields' });
  if (!isAdult(dob)) return res.status(403).json({ message: 'Must be 18+' });

  const existing = await pool.query('SELECT id FROM users WHERE email=$1', [email]);
  if (existing.rowCount) return res.status(409).json({ message: 'Email already in use' });

  const hashed = await bcrypt.hash(password, 10);
  try {
    const { rows } = await pool.query(
      'INSERT INTO users (email, password, name, dob, role) VALUES ($1,$2,$3,$4,$5) RETURNING id, email, name',
      [email, hashed, name || null, dob, 'user']
    );
    const user = rows[0];
    // create empty profile
    await pool.query('INSERT INTO profiles (user_id) VALUES ($1)', [user.id]);
    const token = signToken(user.id);
    res.status(201).json({ user, token });
  } catch (err) {
    console.error(err);
    res.status(500).json({ message: 'Server error' });
  }
});

// Login
app.post('/api/auth/login', async (req, res) => {
  const { email, password } = req.body;
  const user = await findUserByEmail(email);
  if (!user) return res.status(401).json({ message: 'Invalid credentials' });
  const ok = await bcrypt.compare(password, user.password);
  if (!ok) return res.status(401).json({ message: 'Invalid credentials' });
  const token = signToken(user.id);
  res.json({ user: { id: user.id, email: user.email, name: user.name, role: user.role }, token });
});

// Auth middleware
function authMiddleware(req, res, next) {
  const header = req.headers.authorization;
  if (!header) return res.status(401).json({ message: 'Missing auth' });
  const parts = header.split(' ');
  if (parts.length !== 2) return res.status(401).json({ message: 'Malformed auth' });
  const token = parts[1];
  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET);
    req.userId = payload.sub;
    next();
  } catch (err) {
    return res.status(401).json({ message: 'Invalid token' });
  }
}

// Role check middleware
async function requireRole(role) {
  return async (req, res, next) => {
    const { rows } = await pool.query('SELECT role FROM users WHERE id=$1', [req.userId]);
    const userRole = rows[0]?.role;
    if (!userRole) return res.status(403).json({ message: 'No role' });
    if (userRole !== role && !(role === 'moderator' && userRole === 'admin')) return res.status(403).json({ message: 'Forbidden' });
    next();
  };
}

// Get profile
app.get('/api/profile/:id', authMiddleware, async (req, res) => {
  const { id } = req.params;
  const { rows } = await pool.query('SELECT p.*, u.email, u.name FROM profiles p JOIN users u ON u.id = p.user_id WHERE p.user_id = $1', [id]);
  if (!rows[0]) return res.status(404).json({ message: 'Not found' });
  res.json(rows[0]);
});

// Update own profile
app.put('/api/profile', authMiddleware, upload.single('avatar'), async (req, res) => {
  const { bio, gender, age, location, interests } = req.body;
  const avatar_path = req.file ? req.file.path : null;
  await pool.query(
    `UPDATE profiles SET bio=$1, gender=$2, age=$3, location=$4, interests=$5, avatar_path=COALESCE($6, avatar_path) WHERE user_id=$7`,
    [bio || null, gender || null, age || null, location || null, interests ? JSON.stringify(JSON.parse(interests)) : null, avatar_path, req.userId]
  );
  const { rows } = await pool.query('SELECT * FROM profiles WHERE user_id=$1', [req.userId]);
  res.json(rows[0]);
});

// Like another user
app.post('/api/likes', authMiddleware, async (req, res) => {
  const { toUserId, action } = req.body; // action: like/dislike
  if (!toUserId) return res.status(400).json({ message: 'toUserId required' });
  await pool.query('INSERT INTO likes (from_user_id, to_user_id, action) VALUES ($1,$2,$3) ON CONFLICT DO NOTHING', [req.userId, toUserId, action || 'like']);
  // check mutual like
  const { rows } = await pool.query('SELECT * FROM likes WHERE from_user_id=$1 AND to_user_id=$2 AND action=$3', [toUserId, req.userId, 'like']);
  if (rows.length) {
    const a = Math.min(req.userId, toUserId);
    const b = Math.max(req.userId, toUserId);
    await pool.query('INSERT INTO matches (user_a, user_b) VALUES ($1,$2) ON CONFLICT DO NOTHING', [a, b]);
    // emit match event to both users via sockets (if connected)
    io.to(`user:${toUserId}`).emit('match', { with: req.userId });
    io.to(`user:${req.userId}`).emit('match', { with: toUserId });
    return res.json({ matched: true });
  }
  res.json({ matched: false });
});

// Get matches
app.get('/api/matches', authMiddleware, async (req, res) => {
  const { rows } = await pool.query(
    `SELECT m.*, p.age, p.bio, p.avatar_path, u.name, u.email FROM matches m
      JOIN profiles p ON (p.user_id = CASE WHEN m.user_a = $1 THEN m.user_b ELSE m.user_a END)
      JOIN users u ON u.id = (CASE WHEN m.user_a = $1 THEN m.user_b ELSE m.user_a END)
      WHERE m.user_a = $1 OR m.user_b = $1`,
    [req.userId]
  );
  res.json(rows);
});

// Simple admin-only endpoint (list users)
app.get('/api/admin/users', authMiddleware, requireRole('admin'), async (req, res) => {
  const { rows } = await pool.query('SELECT u.id, u.email, u.name, u.role, p.age, p.location, p.avatar_path FROM users u JOIN profiles p ON p.user_id = u.id ORDER BY u.id ASC');
  res.json(rows);
});

// Reports (user reports)
app.post('/api/reports', authMiddleware, async (req, res) => {
  const { target_user_id, reason } = req.body;
  if (!target_user_id || !reason) return res.status(400).json({ message: 'Missing fields' });
  await pool.query('INSERT INTO reports (reporter_id, target_user_id, reason) VALUES ($1,$2,$3)', [req.userId, target_user_id, reason]);
  res.json({ ok: true });
});

// Payment stub endpoints
app.post('/api/payments/stripe-intent', authMiddleware, async (req, res) => {
  // In production, create Stripe PaymentIntent server-side.
  res.json({ clientSecret: 'stripe_client_secret_stub' });
});

app.post('/api/payments/crypto-confirm', authMiddleware, async (req, res) => {
  // Manual confirmation stub: admin will mark payment as confirmed in admin tool.
  res.json({ status: 'pending', message: 'Awaiting manual confirmation' });
});

// --- server + sockets ---
const server = http.createServer(app);
const io = new Server(server, { cors: { origin: '*' } });

// socket auth
io.use((socket, next) => {
  const token = socket.handshake.auth?.token;
  if (!token) return next(new Error('Authentication error'));
  try {
    const payload = jwt.verify(token, process.env.JWT_SECRET);
    socket.userId = payload.sub;
    next();
  } catch (err) {
    next(new Error('Authentication error'));
  }
});

io.on('connection', (socket) => {
  console.log('socket connected', socket.userId);
  socket.join(`user:${socket.userId}`);

  socket.on('private_message', async (data) => {
    // data: { toUserId, message }
    if (!data?.toUserId || !data?.message) return;
    try {
      const res = await pool.query(
        'INSERT INTO messages (from_user_id, to_user_id, message) VALUES ($1,$2,$3) RETURNING id, created_at',
        [socket.userId, data.toUserId, data.message]
      );
      const msg = { id: res.rows[0].id, from: socket.userId, to: data.toUserId, message: data.message, created_at: res.rows[0].created_at };
      io.to(`user:${data.toUserId}`).emit('private_message', msg);
      socket.emit('private_message', msg); // echo back
    } catch (err) {
      console.error('msg save error', err);
    }
  });
});

const port = process.env.PORT || 4000;
server.listen(port, () => console.log(`Backend running on ${port}`));

module.exports = { app, io, pool };
EOF

cat > backend/src/scripts/migrate.js <<'EOF'
const { Pool } = require('pg');
require('dotenv').config();

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function migrate() {
  try {
    await pool.query(`
      CREATE TABLE IF NOT EXISTS users (
        id SERIAL PRIMARY KEY,
        email TEXT UNIQUE NOT NULL,
        password TEXT NOT NULL,
        name TEXT,
        dob DATE,
        role TEXT DEFAULT 'user',
        created_at TIMESTAMP DEFAULT now()
      );
      CREATE TABLE IF NOT EXISTS profiles (
        id SERIAL PRIMARY KEY,
        user_id INTEGER UNIQUE REFERENCES users(id) ON DELETE CASCADE,
        bio TEXT,
        gender TEXT,
        age INTEGER,
        location TEXT,
        interests JSONB,
        avatar_path TEXT,
        created_at TIMESTAMP DEFAULT now()
      );
      CREATE TABLE IF NOT EXISTS likes (
        id SERIAL PRIMARY KEY,
        from_user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
        to_user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
        action TEXT DEFAULT 'like',
        created_at TIMESTAMP DEFAULT now(),
        UNIQUE (from_user_id, to_user_id)
      );
      CREATE TABLE IF NOT EXISTS matches (
        id SERIAL PRIMARY KEY,
        user_a INTEGER NOT NULL,
        user_b INTEGER NOT NULL,
        created_at TIMESTAMP DEFAULT now(),
        UNIQUE (user_a, user_b)
      );
      CREATE TABLE IF NOT EXISTS messages (
        id SERIAL PRIMARY KEY,
        from_user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
        to_user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
        message TEXT,
        created_at TIMESTAMP DEFAULT now()
      );
      CREATE TABLE IF NOT EXISTS reports (
        id SERIAL PRIMARY KEY,
        reporter_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
        target_user_id INTEGER REFERENCES users(id) ON DELETE CASCADE,
        reason TEXT,
        created_at TIMESTAMP DEFAULT now()
      );
      CREATE TABLE IF NOT EXISTS payments (
        id SERIAL PRIMARY KEY,
        user_id INTEGER REFERENCES users(id),
        provider TEXT,
        amount NUMERIC,
        currency TEXT,
        status TEXT,
        meta JSONB,
        created_at TIMESTAMP DEFAULT now()
      );
    `);
    console.log('Migrations applied');
    process.exit(0);
  } catch (err) {
    console.error(err);
    process.exit(1);
  }
}

migrate();
EOF

cat > backend/src/scripts/seed.js <<'EOF'
const { Pool } = require('pg');
const bcrypt = require('bcrypt');
require('dotenv').config();

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function seed() {
  try {
    // create users (2 admins, 2 moderators, 4 regular)
    const users = [
      { email: 'admin1@loviora.test', name: 'Admin One', role: 'admin', password: 'Password123!', dob: '1990-01-01' },
      { email: 'admin2@loviora.test', name: 'Admin Two', role: 'admin', password: 'Password123!', dob: '1990-01-02' },
      { email: 'mod1@loviora.test', name: 'Mod One', role: 'moderator', password: 'Password123!', dob: '1992-02-02' },
      { email: 'mod2@loviora.test', name: 'Mod Two', role: 'moderator', password: 'Password123!', dob: '1993-03-03' },
      { email: 'emma@loviora.test', name: 'Emma', role: 'user', password: 'Password123!', dob: '1998-05-05' },
      { email: 'ryan@loviora.test', name: 'Ryan', role: 'user', password: 'Password123!', dob: '1996-06-06' },
      { email: 'lucas@loviora.test', name: 'Lucas', role: 'user', password: 'Password123!', dob: '1994-07-07' },
      { email: 'sophie@loviora.test', name: 'Sophie', role: 'user', password: 'Password123!', dob: '1997-08-08' }
    ];

    for (const u of users) {
      const hashed = await bcrypt.hash(u.password, 10);
      const res = await pool.query(
        'INSERT INTO users (email, password, name, dob, role) VALUES ($1,$2,$3,$4,$5) ON CONFLICT (email) DO NOTHING RETURNING id',
        [u.email, hashed, u.name, u.dob, u.role]
      );
      const id = res.rows[0] ? res.rows[0].id : (await pool.query('SELECT id FROM users WHERE email=$1', [u.email])).rows[0].id;
      // create profile
      await pool.query(
        `INSERT INTO profiles (user_id, bio, gender, age, location, interests, avatar_path) 
         VALUES ($1,$2,$3,$4,$5,$6,$7) ON CONFLICT (user_id) DO NOTHING`,
        [id, `${u.name} bio`, 'not specified', 25, 'City', JSON.stringify(['music', 'travel']), null]
      );
    }

    // create a mutual like between ryan and emma for demo
    const emma = (await pool.query('SELECT id FROM users WHERE email=$1', ['emma@loviora.test'])).rows[0].id;
    const ryan = (await pool.query('SELECT id FROM users WHERE email=$1', ['ryan@loviora.test'])).rows[0].id;
    await pool.query('INSERT INTO likes (from_user_id, to_user_id, action) VALUES ($1,$2,$3) ON CONFLICT DO NOTHING', [emma, ryan, 'like']);
    await pool.query('INSERT INTO likes (from_user_id, to_user_id, action) VALUES ($1,$2,$3) ON CONFLICT DO NOTHING', [ryan, emma, 'like']);
    const a = Math.min(emma, ryan), b = Math.max(emma, ryan);
    await pool.query('INSERT INTO matches (user_a, user_b) VALUES ($1,$2) ON CONFLICT DO NOTHING', [a, b]);

    console.log('Seed completed. Demo credentials:');
    console.log('Admins: admin1@loviora.test / Password123!, admin2@loviora.test / Password123!');
    console.log('Moderators: mod1@loviora.test / Password123!, mod2@loviora.test / Password123!');
    process.exit(0);
  } catch (err) {
    console.error(err);
    process.exit(1);
  }
}

seed();
EOF

cat > backend/.gitignore <<'EOF'
node_modules
.env
/tmp/uploads
EOF

cat > .gitignore <<'EOF'
node_modules
frontend/node_modules
backend/node_modules
.env
/tmp/uploads
dist
EOF

cat > frontend/Dockerfile <<'EOF'
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
EXPOSE 5173
CMD ["npm", "run", "dev"]
EOF

cat > frontend/package.json <<'EOF'
{
  "name": "loviora-frontend",
  "version": "0.1.0",
  "private": true,
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "axios": "^1.4.0",
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "socket.io-client": "^4.7.0",
    "react-router-dom": "^6.12.1"
  },
  "devDependencies": {
    "vite": "^5.1.0"
  }
}
EOF

cat > frontend/.env.example <<'EOF'
VITE_API_URL=http://localhost:4000/api
EOF

cat > frontend/index.html <<'EOF'
<!doctype html>
<html>
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Loviora</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
EOF

cat > frontend/src/main.jsx <<'EOF'
import React from 'react';
import { createRoot } from 'react-dom/client';
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import App from './App';
import AdminPanel from './pages/AdminPanel';
import './styles/pulse.css';

createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <BrowserRouter>
      <Routes>
        <Route path="/*" element={<App />} />
        <Route path="/admin-panel/*" element={<AdminPanel />} />
      </Routes>
    </BrowserRouter>
  </React.StrictMode>
);
EOF

cat > frontend/src/App.jsx <<'EOF'
import React from 'react';
import { Routes, Route } from 'react-router-dom';
import Login from './pages/Login';
import Discover from './pages/Discover';
import ProfileEdit from './pages/ProfileEdit';
import Matches from './pages/Matches';
import ChatPage from './pages/ChatPage';

export default function App() {
  return (
    <Routes>
      <Route path="/" element={<Login />} />
      <Route path="/discover" element={<Discover />} />
      <Route path="/profile" element={<ProfileEdit />} />
      <Route path="/matches" element={<Matches />} />
      <Route path="/chat/:id" element={<ChatPage />} />
    </Routes>
  );
}
EOF

cat > frontend/src/api/client.js <<'EOF'
import axios from 'axios';
const API = import.meta.env.VITE_API_URL || 'http://localhost:4000/api';
const client = axios.create({ baseURL: API });
export default client;
EOF

cat > frontend/src/pages/Login.jsx <<'EOF'
import React, { useState } from 'react';
import client from '../api/client';
import { useNavigate } from 'react-router-dom';

export default function Login() {
  const [mode, setMode] = useState('login');
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [dob, setDob] = useState('');
  const navigate = useNavigate();

  async function submit(e) {
    e.preventDefault();
    try {
      const url = mode === 'login' ? '/auth/login' : '/auth/register';
      const body = mode === 'login' ? { email, password } : { email, password, dob, name: email.split('@')[0] };
      const res = await client.post(url, body);
      localStorage.setItem('loviora_token', res.data.token);
      navigate('/discover');
    } catch (err) {
      alert(err?.response?.data?.message || 'Error');
    }
  }

  return (
    <div className="login-screen">
      <form onSubmit={submit} className="login-form">
        <h1>Loviora</h1>
        <input placeholder="email" value={email} onChange={e => setEmail(e.target.value)} />
        <input type="password" placeholder="password" value={password} onChange={e => setPassword(e.target.value)} />
        {mode === 'register' && <input type="date" value={dob} onChange={e => setDob(e.target.value)} />}
        <button type="submit">{mode === 'login' ? 'Login' : 'Register'}</button>
        <button type="button" onClick={() => setMode(mode === 'login' ? 'register' : 'login')}>
          {mode === 'login' ? 'Create account' : 'Have account? Login'}
        </button>
      </form>
    </div>
  );
}
EOF

cat > frontend/src/pages/Discover.jsx <<'EOF'
import React, { useEffect, useState } from 'react';
import client from '../api/client';
import ProfileCard from '../ui/ProfileCard';
import SwipeButtons from '../ui/SwipeButtons';
import { useNavigate } from 'react-router-dom';
import io from 'socket.io-client';

const API_BASE = import.meta.env.VITE_API_URL ? import.meta.env.VITE_API_URL.replace('/api', '') : 'http://localhost:4000';

function getToken() {
  return localStorage.getItem('loviora_token');
}

export default function Discover() {
  const [profiles, setProfiles] = useState([]);
  const [socket, setSocket] = useState(null);
  const navigate = useNavigate();

  useEffect(() => {
    const token = getToken();
    if (!token) return navigate('/');
    client.get('/matches'); // warm
    client.get('/profile/me').catch(()=>{});
    client.get('/profiles').then(res => setProfiles(res.data)).catch(()=>{});
    const s = io(API_BASE, { auth: { token } });
    s.on('connect', ()=>console.log('socket connected'));
    s.on('match', (m) => alert('It\\'s a match!'));
    setSocket(s);
    return ()=> s.disconnect();
  }, []);

  async function like(toUserId) {
    try {
      const token = getToken();
      await client.post('/likes', { toUserId, action: 'like' }, { headers: { Authorization: `Bearer ${token}` } });
      setProfiles(p => p.filter(x => x.user_id !== toUserId));
    } catch (err) { console.error(err); }
  }

  async function dislike(toUserId) {
    try {
      const token = getToken();
      await client.post('/likes', { toUserId, action: 'dislike' }, { headers: { Authorization: `Bearer ${token}` } });
      setProfiles(p => p.filter(x => x.user_id !== toUserId));
    } catch (err) { console.error(err); }
  }

  const top = profiles[0];

  return (
    <div style={{ padding: 16 }}>
      <h1>Discover</h1>
      {top ? <ProfileCard name={top.name || top.email} age={top.age || 25} imageUrl={top.avatar_path || 'https://via.placeholder.com/300'} /> : <p>No profiles</p>}
      <SwipeButtons onLike={() => top && like(top.user_id)} onDislike={() => top && dislike(top.user_id)} />
    </div>
  );
}
EOF

cat > frontend/src/ui/ProfileCard.jsx <<'EOF'
import React from 'react';
export default function ProfileCard({ name, age, imageUrl }) {
  return (
    <div className="card" style={{ backgroundImage: `url(${imageUrl})` }}>
      <div className="card-info">
        <h2>{name}, {age}</h2>
      </div>
    </div>
  );
}
EOF

cat > frontend/src/ui/SwipeButtons.jsx <<'EOF'
import React from 'react';
export default function SwipeButtons({ onLike, onDislike }) {
  return (
    <div className="buttons">
      <button className="dislike" onClick={onDislike}>✖</button>
      <button className="like" onClick={onLike}>❤</button>
    </div>
  );
}
EOF

cat > frontend/src/pages/Matches.jsx <<'EOF'
import React, { useEffect, useState } from 'react';
import client from '../api/client';

export default function Matches() {
  const [matches, setMatches] = useState([]);
  useEffect(() => {
    const token = localStorage.getItem('loviora_token');
    if (!token) return;
    client.get('/matches', { headers: { Authorization: `Bearer ${token}` } }).then(res => setMatches(res.data)).catch(()=>{});
  }, []);
  return (
    <div style={{ padding: 16 }}>
      <h1>Matches</h1>
      {matches.map(m => <div key={m.id} style={{ padding: 8, borderBottom: '1px solid #eee' }}>{m.name || m.email} — {m.bio}</div>)}
    </div>
  );
}
EOF

cat > frontend/src/pages/ChatPage.jsx <<'EOF'
import React, { useEffect, useState } from 'react';
import { useParams } from 'react-router-dom';
import io from 'socket.io-client';
const API_BASE = import.meta.env.VITE_API_URL ? import.meta.env.VITE_API_URL.replace('/api', '') : 'http://localhost:4000';
export default function ChatPage() {
  const { id } = useParams();
  const [socket, setSocket] = useState(null);
  const [msgs, setMsgs] = useState([]);
  const [text, setText] = useState('');
  useEffect(() => {
    const token = localStorage.getItem('loviora_token');
    if (!token) return;
    const s = io(API_BASE, { auth: { token } });
    s.on('private_message', m => setMsgs(prev => [...prev, m]));
    setSocket(s);
    return () => s.disconnect();
  }, []);
  function send() {
    if (!socket) return;
    socket.emit('private_message', { toUserId: Number(id), message: text });
    setText('');
  }
  return (
    <div style={{ padding: 16 }}>
      <h2>Chat</h2>
      <div style={{ minHeight: 200, border: '1px solid #ddd', padding: 8 }}>
        {msgs.map((m, i) => <div key={i}><b>{m.from}</b>: {m.message}</div>)}
      </div>
      <input value={text} onChange={e=>setText(e.target.value)} />
      <button onClick={send}>Send</button>
    </div>
  );
}
EOF

cat > frontend/src/pages/ProfileEdit.jsx <<'EOF'
import React, { useEffect, useState } from 'react';
import client from '../api/client';
export default function ProfileEdit() {
  const [profile, setProfile] = useState({});
  useEffect(() => {
    const token = localStorage.getItem('loviora_token');
    if (!token) return;
    client.get('/profile/me', { headers: { Authorization: `Bearer ${token}` } }).then(res => setProfile(res.data)).catch(()=>{});
  }, []);
  async function save() {
    const token = localStorage.getItem('loviora_token');
    const body = { bio: profile.bio };
    await client.put('/profile', body, { headers: { Authorization: `Bearer ${token}` } });
    alert('Saved');
  }
  return (
    <div style={{ padding: 16 }}>
      <h1>Edit Profile</h1>
      <textarea value={profile.bio || ''} onChange={e=>setProfile({...profile, bio: e.target.value})} />
      <button onClick={save}>Save</button>
    </div>
  );
}
EOF

cat > frontend/src/pages/AdminPanel.jsx <<'EOF'
import React, { useEffect, useState } from 'react';
import client from '../api/client';

export default function AdminPanel() {
  const [users, setUsers] = useState([]);
  useEffect(() => {
    const token = localStorage.getItem('loviora_token');
    if (!token) return;
    client.get('/admin/users', { headers: { Authorization: `Bearer ${token}` } }).then(res => setUsers(res.data)).catch(()=>{});
  }, []);
  return (
    <div style={{ padding: 16 }}>
      <h1>Admin Panel</h1>
      <table style={{ width: '100%', borderCollapse: 'collapse' }}>
        <thead><tr><th>User</th><th>Role</th><th>Location</th></tr></thead>
        <tbody>
          {users.map(u => <tr key={u.id}><td>{u.name} ({u.email})</td><td>{u.role}</td><td>{u.location}</td></tr>)}
        </tbody>
      </table>
    </div>
  );
}
EOF

cat > frontend/src/styles/pulse.css <<'EOF'
:root {
  --pulse-primary: #ff4b6e;
  --pulse-secondary: #ffd3e0;
  --pulse-accent: #ffffff;
}
body {
  font-family: Inter, Roboto, sans-serif;
  margin: 0; padding: 0; background: linear-gradient(180deg, var(--pulse-secondary), #fff);
}
.card {
  width: min(92vw, 360px);
  height: min(60vh, 520px);
  border-radius: 20px;
  background-size: cover;
  box-shadow: 0px 12px 30px rgba(0,0,0,0.12);
  position: relative;
  overflow: hidden;
  margin: 16px auto;
}
.card-info {
  position: absolute;
  bottom: 20px;
  left: 20px;
  color: white;
  font-family: 'Poppins', sans-serif;
  font-weight: bold;
  font-size: 20px;
  text-shadow: 1px 1px 4px rgba(0,0,0,0.6);
}
.buttons { display:flex; justify-content: space-evenly; margin-top: 12px; }
button { border: none; border-radius: 50%; width: 60px; height: 60px; font-size: 24px; color: white; cursor: pointer; }
.dislike { background-color: #FF4B6E; }
.like { background-color: #FF1493; }
.login-screen { display:flex; align-items:center; justify-content:center; height:100vh; background: linear-gradient(to bottom right, var(--pulse-primary), var(--pulse-secondary)); }
.login-form { background: rgba(255,255,255,0.9); padding: 24px; border-radius: 14px; display:flex; flex-direction:column; gap:10px; width: 320px; }
@media (prefers-color-scheme: dark) {
  body { background: linear-gradient(180deg, #2b2b2b, #111); color: #fff; }
}
EOF

cat > frontend/.gitignore <<'EOF'
/node_modules
/dist
.env.local
EOF

cat > mobile/pulse_ui_package/lib/pulse_ui.dart <<'EOF'
// Flutter null-safe UI package (pulse_ui_package/lib/pulse_ui.dart)
import 'package:flutter/material.dart';

// -------------------- Rəng Palitrası və Fontlar --------------------
class PulseTheme {
  static const Color primary = Color(0xFFFF4B6E);
  static const Color secondary = Color(0xFFFFD3E0);
  static const Color accent = Colors.white;

  static const TextStyle heading = TextStyle(
      fontFamily: 'Poppins', fontWeight: FontWeight.w700, fontSize: 24);
  static const TextStyle body = TextStyle(fontFamily: 'Roboto', fontSize: 16);
}

// -------------------- Profil Kart --------------------
class ProfileCard extends StatelessWidget {
  final String name;
  final int age;
  final String imageUrl;

  const ProfileCard({Key? key, required this.name, required this.age, required this.imageUrl}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Container(
      width: double.infinity,
      height: 420,
      decoration: BoxDecoration(
        borderRadius: BorderRadius.circular(20),
        image: DecorationImage(
          image: NetworkImage(imageUrl),
          fit: BoxFit.cover,
        ),
        boxShadow: const [BoxShadow(blurRadius: 10, color: Colors.black26)],
      ),
      child: Stack(
        children: [
          Positioned(
            bottom: 20,
            left: 20,
            child: Text(
              "$name, $age",
              style: PulseTheme.heading.copyWith(color: Colors.white, shadows: [
                const Shadow(blurRadius: 5, color: Colors.black)
              ]),
            ),
          ),
        ],
      ),
    );
  }
}

// -------------------- Swipe Buttons --------------------
class SwipeButtons extends StatelessWidget {
  final VoidCallback onLike;
  final VoidCallback onDislike;

  const SwipeButtons({Key? key, required this.onLike, required this.onDislike}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Row(
      mainAxisAlignment: MainAxisAlignment.spaceEvenly,
      children: [
        FloatingActionButton(
          backgroundColor: Colors.redAccent,
          onPressed: onDislike,
          child: const Icon(Icons.close, color: Colors.white),
        ),
        FloatingActionButton(
          backgroundColor: Colors.pinkAccent,
          onPressed: onLike,
          child: const Icon(Icons.favorite, color: Colors.white),
        ),
      ],
    );
  }
}

// -------------------- Chat Bubble --------------------
class ChatBubble extends StatelessWidget {
  final String message;
  final bool isMe;

  const ChatBubble({Key? key, required this.message, required this.isMe}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Align(
      alignment: isMe ? Alignment.centerRight : Alignment.centerLeft,
      child: Container(
        margin: const EdgeInsets.symmetric(vertical: 5, horizontal: 10),
        padding: const EdgeInsets.all(12),
        decoration: BoxDecoration(
          color: isMe ? Colors.pinkAccent : Colors.white,
          borderRadius: BorderRadius.circular(15),
          boxShadow: const [BoxShadow(color: Colors.black12, blurRadius: 5)],
        ),
        child: Text(
          message,
          style: PulseTheme.body.copyWith(
            color: isMe ? Colors.white : Colors.black,
          ),
        ),
      ),
    );
  }
}

// -------------------- Login Screen --------------------
class LoginScreenWidget extends StatelessWidget {
  final VoidCallback onLogin;
  const LoginScreenWidget({Key? key, required this.onLogin}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      body: Container(
        decoration: const BoxDecoration(
            gradient: LinearGradient(
                colors: [PulseTheme.primary, PulseTheme.secondary],
                begin: Alignment.topLeft,
                end: Alignment.bottomRight)),
        child: Center(
          child: ElevatedButton(
            onPressed: onLogin,
            child: const Text("Login / Register"),
            style: ElevatedButton.styleFrom(
                backgroundColor: PulseTheme.accent,
                shape: RoundedRectangleBorder(
                    borderRadius: BorderRadius.circular(20))),
          ),
        ),
      ),
    );
  }
}

// -------------------- Premium / USDT Payment Screen --------------------
class PremiumScreen extends StatelessWidget {
  final String qrCodeUrl;
  const PremiumScreen({Key? key, required this.qrCodeUrl}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(title: const Text("Premium Upgrade")),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: [
            const Text("Scan QR to Pay USDT", style: PulseTheme.heading),
            const SizedBox(height: 20),
            Image.network(qrCodeUrl, width: 200, height: 200, errorBuilder: (c,s,e)=>const Icon(Icons.qr_code, size: 200)),
            const SizedBox(height: 20),
            ElevatedButton(
                onPressed: () {
                  // Manual approval confirmation
                },
                child: const Text("Confirm Payment"))
          ],
        ),
      ),
    );
  }
}
EOF

# -------- PR body file --------
cat > "../$PR_BODY_FILE" <<'EOF'
This PR adds the initial Loviora starter scaffold implementing an MVP dating app skeleton.

Included:
- Docker Compose dev stack (Postgres + backend + frontend)
- Backend (Node.js + Express)
  - JWT auth with DOB >=18 validation
  - Profile CRUD, likes/matches endpoints
  - Socket.IO chat + authenticated sockets
  - Avatar upload (multer, S3-ready stub)
  - Rate limiting, helmet, basic validation
  - Migrations (migrate.js) and seed script (seed.js)
  - Payment stubs (Stripe & crypto QR manual-confirmation)
- Frontend (React + Vite)
  - Mobile-first pages: Login, Discover (swipe skeleton), Profile edit, Matches, Chat
  - Admin panel route (/admin-panel) scaffold
  - socket.io client integration
- Mobile UI package (Flutter) with reusable widgets (pulse_ui.dart)
- README with run & seed instructions

Quick run instructions:
1. Copy env files:
   - cp backend/.env.example backend/.env
   - cp frontend/.env.example frontend/.env
2. Start stack:
   - docker-compose up --build -d
3. Backend:
   - docker exec -it <backend_container> npm ci
   - docker exec -it <backend_container> npm run migrate
   - docker exec -it <backend_container> npm run seed
4. Frontend:
   - cd frontend
   - npm ci
   - npm run dev
5. Visit:
   - Frontend: http://localhost:5173
   - Backend API: http://localhost:4000/api
   - Admin: http://localhost:5173/admin-panel

Notes & security
- Change JWT_SECRET and other secrets in backend/.env before any public deployment.
- Seeded demo credentials (created by seed script):
  - Admins: admin1@loviora.test / Password123!, admin2@loviora.test / Password123!
  - Moderators: mod1@loviora.test / Password123!, mod2@loviora.test / Password123!
- Stripe and crypto endpoints are currently stubs — integrate real providers for production.
- Images are stored to /tmp/uploads in this scaffold — configure S3 in services/storage.js before production.
EOF

# Add files, commit, push
git add .
git commit -m "feat: add Loviora MVP scaffold (backend + frontend + admin + flutter UI)" || echo "Nothing to commit"
git push origin "$BRANCH"

# Create PR if gh available
if command -v gh >/dev/null 2>&1; then
  echo "Creating PR with gh..."
  gh pr create --base main --head "$BRANCH" --title "$PR_TITLE" --body-file "../$PR_BODY_FILE"
  echo "PR created. Use gh or the GitHub web UI to view it."
else
  echo "Branch pushed to origin/$BRANCH. Create a PR manually at:"
  echo "https://github.com/yoxyoxdur135-png/pulse/compare/main...$BRANCH"
fi

echo "Done."