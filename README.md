# Brain
BRAIN 🧠 – The creator-first social network | Get paid to post • Reels • Live • Verified Checkmarks • Built-in AI (CHIBEST) • Real Stripe paymentsBRAIN – Get paid to post. Reels, Live, AI Chatbot, Verified &amp; VIP badges.
require('dotenv').config();
const express = require('express');
const cors = require('cors');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');
const sqlite3 = require('sqlite3').verbose();
const multer = require('multer');
const { CloudinaryStorage } = require('multer-storage-cloudinary');
const { v2: cloudinary } = require('cloudinary');
const stripe = require('stripe')(process.env.STRIPE_SECRET_KEY);
const http = require('http');
const { Server } = require('socket.io');

cloudinary.config({
  cloud_name: process.env.CLOUDINARY_CLOUD_NAME,
  api_key: process.env.CLOUDINARY_API_KEY,
  api_secret: process.env.CLOUDINARY_API_SECRET,
});

const app = express();
const server = http.createServer(app);
const io = new Server(server, { cors: { origin: "*" } });

app.use(cors());
app.use(express.json());
app.use(express.raw({ type: 'application/json' }));

const db = new sqlite3.Database('./brain.db', (err) => {
  if (err) console.error(err);
  console.log('BRAIN Database Connected');
});

db.serialize(() => {
  db.run(`CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT UNIQUE,
    email TEXT UNIQUE,
    password TEXT,
    avatar TEXT DEFAULT 'https://via.placeholder.com/150',
    verified INTEGER DEFAULT 0,
    vip INTEGER DEFAULT 0,
    isProfessional INTEGER DEFAULT 0
  )`);
  db.run(`CREATE TABLE IF NOT EXISTS posts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    userId INTEGER,
    content TEXT,
    image TEXT,
    isPaid INTEGER DEFAULT 0,
    price INTEGER DEFAULT 999,
    views INTEGER DEFAULT 0,
    createdAt DATETIME DEFAULT CURRENT_TIMESTAMP
  )`);
  db.run(`CREATE TABLE IF NOT EXISTS live_streams (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    creatorId INTEGER,
    title TEXT,
    isLive INTEGER DEFAULT 1,
    viewerCount INTEGER DEFAULT 0
  )`);
});

// Authentication middleware
const authenticate = (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: "No token" });
  jwt.verify(token, 'brainsecret2025', (err, user) => {
    if (err) return res.status(403).json({ error: "Invalid token" });
    req.user = user;
    next();
  });
};

// Routes
app.post('/api/register', async (req, res) => {
  const { username, email, password } = req.body;
  const hashed = await bcrypt.hash(password, 10);
  db.run('INSERT INTO users (username, email, password) VALUES (?, ?, ?)', [username, email, hashed], function(err) {
    if (err) return res.status(400).json({ error: "User exists" });
    res.json({ id: this.lastID, username });
  });
});

app.post('/api/login', (req, res) => {
  const { email, password } = req.body;
  db.get('SELECT * FROM users WHERE email = ?', [email], async (err, user) => {
    if (!user || !await bcrypt.compare(password, user.password)) return res.status(401).json({ error: "Wrong credentials" });
    const token = jwt.sign({ id: user.id }, 'brainsecret2025', { expiresIn: '30d' });
    res.json({ token, user: { id: user.id, username: user.username, verified: user.verified, vip: user.vip, isProfessional: user.isProfessional } });
  });
});

// CHIBEST AI (placeholder – works instantly)
app.post('/api/chibest', authenticate, (req, res) => {
  const { message } = req.body;
  res.json({ reply: `CHIBEST here! You said: "${message}". Let’s go viral ` });
});

// Toggle Professional Mode
app.post('/api/toggle-professional', authenticate, (req, res) => {
  db.run(`UPDATE users SET isProfessional = NOT isProfessional WHERE id = ?`, [req.user.id], function() {
    db.get(`SELECT isProfessional FROM users WHERE id = ?`, [req.user.id], (err, row) => {
      res.json({ isProfessional: row.isProfessional });
    });
  });
});

io.on('connection', (socket) => {
  console.log('User connected to BRAIN Live');
  socket.on('join-stream', (id) => socket.join(`stream_${id}`));
});

server.listen(process.env.PORT || 5000, () => {
  console.log('BRAIN SERVER RUNNING ');
});
