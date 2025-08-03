# perfume-business-app
Backend/Package.Json 

{
  "name": "perfume-business-backend",
  "version": "1.0.0",
  "description": "Backend API for Perfume Business App",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "bcryptjs": "^2.4.3",
    "cors": "^2.8.5",
    "dotenv": "^16.3.1",
    "express": "^4.18.2",
    "jsonwebtoken": "^9.0.2",
    "mongoose": "^8.3.2"
  },
  "devDependencies": {
    "nodemon": "^3.1.0"
  },
  "author": "",
  "license": "MIT"
}

Backend/app.js

const express = require('express');
const cors = require('cors');
const mongoose = require('mongoose');
const dotenv = require('dotenv');

dotenv.config();

const authRoutes = require('./routes/auth');
const perfumeRoutes = require('./routes/perfume');

const app = express();

app.use(cors());
app.use(express.json());

app.use('/api/auth', authRoutes);
app.use('/api/perfumes', perfumeRoutes);

app.get('/', (req, res) => {
  res.send('Perfume Business API is running!');
});

module.exports = app;

backend/server.js

const app = require('./app');
const mongoose = require('mongoose');

const PORT = process.env.PORT || 5000;
const MONGO_URI = process.env.MONGO_URI || 'mongodb://localhost:27017/perfume-business';

mongoose.connect(MONGO_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true,
}).then(() => {
  console.log('MongoDB connected!');
  app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
  });
}).catch(err => {
  console.error('MongoDB connection error:', err);
});

backend/.env.example
PORT=5000
MONGO_URI=mongodb://localhost:27017/perfume-business
JWT_SECRET=your_jwt_secret

backend/model/User.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const UserSchema = new mongoose.Schema({
  username: { type: String, required: true, unique: true, minlength: 3 },
  email: { type: String, required: true, unique: true, match: /.+\@.+\..+/ },
  password: { type: String, required: true, minlength: 6 },
  isAdmin: { type: Boolean, default: false }
});

// Hash password before saving
UserSchema.pre('save', async function (next) {
  if (!this.isModified('password')) return next();
  this.password = await bcrypt.hash(this.password, 10);
  next();
});

// Compare password
UserSchema.methods.comparePassword = async function (candidatePassword) {
  return bcrypt.compare(candidatePassword, this.password);
};

module.exports = mongoose.model('User', UserSchema);


const mongoose = require('mongoose');

const PerfumeSchema = new mongoose.Schema({
  name: { type: String, required: true },
  brand: { type: String, required: true },
  price: { type: Number, required: true },
  description: { type: String },
  image: { type: String }
}, { timestamps: true });

module.exports = mongoose.model('Perfume', PerfumeSchema);


const express = require('express');
const router = express.Router();
const User = require('../models/User');
const jwt = require('jsonwebtoken');

// Register
router.post('/register', async (req, res) => {
  try {
    const { username, email, password } = req.body;
    const existing = await User.findOne({ email });
    if (existing) return res.status(400).json({ message: 'Email already exists' });
    const user = new User({ username, email, password });
    await user.save();
    res.status(201).json({ message: 'User registered successfully' });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// Login
router.post('/login', async (req, res) => {
  try {
    const { email, password } = req.body;
    const user = await User.findOne({ email });
    if (!user || !(await user.comparePassword(password)))
      return res.status(401).json({ message: 'Invalid credentials' });

    const token = jwt.sign(
      { id: user._id, username: user.username, isAdmin: user.isAdmin },
      process.env.JWT_SECRET,
      { expiresIn: '1d' }
    );
    res.json({ token, user: { id: user._id, username: user.username, isAdmin: user.isAdmin } });
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

module.exports = router;


const express = require('express');
const router = express.Router();
const Perfume = require('../models/Perfume');
const jwt = require('jsonwebtoken');

// Middleware for authentication
function verifyToken(req, res, next) {
  const bearer = req.headers['authorization'];
  if (!bearer) return res.status(401).json({ message: 'Unauthorized' });
  const token = bearer.split(' ')[1];
  jwt.verify(token, process.env.JWT_SECRET, (err, user) => {
    if (err) return res.status(403).json({ message: 'Forbidden' });
    req.user = user;
    next();
  });
}

// Get all perfumes
router.get('/', async (req, res) => {
  try {
    const perfumes = await Perfume.find();
    res.json(perfumes);
  } catch (err) {
    res.status(500).json({ message: err.message });
  }
});

// Add new perfume (admin only)
router.post('/', verifyToken, async (req, res) => {
  if (!req.user.isAdmin)
    return res.status(403).json({ message: 'Only admin can add perfumes' });
  try {
    const perfume = new Perfume(req.body);
    await perfume.save();
    res.status(201).json(perfume);
  } catch (err) {
    res.status(400).json({ message: err.message });
  }
});

module.exports = router;
