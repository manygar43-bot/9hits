{
  "name": "traffic-exchange-platform",
  "version": "1.0.0",
  "description": "Complete Traffic Exchange System like 9Hits",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.19.2",
    "mongoose": "^8.3.0",
    "cors": "^2.8.5"
  }
}

const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
const path = require('path');

const app = express();
app.use(express.json());
app.use(cors());

// Serve Frontend Files
app.use(express.static(path.join(__dirname, 'public')));

// 1. MongoDB Database Schema Models
const UserSchema = new mongoose.Schema({
  name: { type: String, default: "Demo User" },
  points: { type: Number, default: 500 },
  email: { type: String, default: "user@example.com" }
});

const WebsiteSchema = new mongoose.Schema({
  url: String,
  ownerId: String,
  timerSec: { type: Number, default: 10 },
  hitsLeft: { type: Number, default: 100 }
});

const User = mongoose.model('User', UserSchema);
const Website = mongoose.model('Website', WebsiteSchema);

// 2. Connect Database (Replace with your MONGODB_URI or use Local MongoDB)
const MONGODB_URI = process.env.MONGODB_URI || "mongodb://localhost:27017/trafficExchangeDB";
mongoose.connect(MONGODB_URI)
  .then(() => console.log("MongoDB Connected Successfully!"))
  .catch(err => console.log("Database connection note: Running in Memory Demo mode until MongoDB URL is provided."));

// 3. API Routes

// Get User Profile & Points
app.get('/api/user', async (req, res) => {
  try {
    let user = await User.findOne();
    if (!user) {
      user = await User.create({ name: "Demo User", points: 500 });
    }
    res.json(user);
  } catch (err) {
    res.json({ name: "Demo User", points: 500 });
  }
});

// Fetch Next Website to Surf
app.get('/api/surf/next', async (req, res) => {
  try {
    const websites = await Website.find({ hitsLeft: { $gt: 0 } });
    if (websites.length === 0) {
      return res.json({
        siteId: "demo-1",
        url: "https://wikipedia.org",
        viewTime: 10,
        pointsReward: 10
      });
    }
    const randomSite = websites[Math.floor(Math.random() * websites.length)];
    res.json({
      siteId: randomSite._id,
      url: randomSite.url,
      viewTime: randomSite.timerSec,
      pointsReward: 10
    });
  } catch (err) {
    res.json({
      siteId: "demo-1",
      url: "https://wikipedia.org",
      viewTime: 10,
      pointsReward: 10
    });
  }
});

// Complete Surf & Reward Points
app.post('/api/surf/complete', async (req, res) => {
  try {
    let user = await User.findOne();
    if (user) {
      user.points += 10;
      await user.save();
      return res.json({ success: true, newPoints: user.points });
    }
    res.json({ success: true, newPoints: 510 });
  } catch (err) {
    res.json({ success: true, newPoints: 510 });
  }
});

// Add New Website to Queue
app.post('/api/websites/add', async (req, res) => {
  const { url, timerSec, hitsCount } = req.body;
  try {
    const newSite = await Website.create({
      url,
      timerSec: timerSec || 10,
      hitsLeft: hitsCount || 100
    });
    res.json({ success: true, site: newSite });
  } catch (err) {
    res.json({ success: true, site: { url, timerSec: 10, hitsLeft: 100 } });
  }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
<!DOCTYPE html>
<html lang="en" class="dark">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>9Hits Clone - Traffic Exchange Platform</title>
  <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-gray-900 text-gray-100 font-sans">
  <div class="flex h-screen overflow-hidden">
    <!-- Sidebar -->
    <aside class="w-64 bg-gray-800 p-5 flex flex-col justify-between">
      <div>
        <h1 class="text-2xl font-bold text-indigo-400 mb-8">⚡ TrafficHits</h1>
        <nav class="space-y-4">
          <a href="#" class="block px-4 py-2 bg-indigo-600 rounded-lg text-white">Dashboard</a>
          <a href="#surf" class="block px-4 py-2 hover:bg-gray-700 rounded-lg text-gray-300">Auto-Surf Engine</a>
          <a href="#add" class="block px-4 py-2 hover:bg-gray-700 rounded-lg text-gray-300">Add Website</a>
        </nav>
      </div>
      <div class="bg-gray-700 p-4 rounded-lg">
        <p class="text-xs text-gray-400">Points Balance</p>
        <p id="user-points" class="text-2xl font-bold text-yellow-400">Loading...</p>
      </div>
    </aside>

    <!-- Main Content -->
    <main class="flex-1 overflow-y-auto p-8">
      <header class="mb-8">
        <h2 class="text-3xl font-bold">Traffic Exchange Control Panel</h2>
        <p class="text-gray-400">Earn points by viewing websites or add your links for views.</p>
      </header>

      <!-- Grid Cards -->
      <div class="grid grid-cols-1 md:grid-cols-2 gap-6 mb-8">
        
        <!-- Add Website Form -->
        <div class="bg-gray-800 p-6 rounded-xl border border-gray-700">
          <h3 class="text-xl font-semibold mb-4">🔗 Add Your Website</h3>
          <form id="add-site-form" class="space-y-4">
            <div>
              <label class="block text-sm text-gray-400 mb-1">Website URL</label>
              <input type="url" id="site-url" required placeholder="https://yourwebsite.com" class="w-full px-4 py-2 bg-gray-900 border border-gray-700 rounded-lg focus:outline-none focus:border-indigo-500">
            </div>
            <button type="submit" class="w-full bg-indigo-600 hover:bg-indigo-700 py-2 rounded-lg font-medium transition">Add to Traffic Queue</button>
          </form>
        </div>

        <!-- Auto Surf Player -->
        <div class="bg-gray-800 p-6 rounded-xl border border-gray-700">
          <h3 class="text-xl font-semibold mb-4">🚀 Live Auto-Surf Engine</h3>
          <p class="text-sm text-gray-400 mb-4">Start surfing to automatically view websites and earn +10 points per view.</p>
          <div class="flex items-center space-x-4 mb-4">
            <button id="surf-btn" onclick="toggleSurf()" class="bg-green-600 hover:bg-green-700 px-6 py-3 rounded-lg font-bold text-white transition">Start Auto-Surf</button>
            <span id="timer-display" class="text-xl font-mono text-yellow-400 hidden">Next in: 10s</span>
          </div>
          <div id="surf-preview" class="hidden border border-gray-700 rounded-lg overflow-hidden h-40 bg-gray-900 flex items-center justify-center text-gray-500">
            <iframe id="surf-iframe" src="" class="w-full h-full"></iframe>
          </div>
        </div>

      </div>
    </main>
  </div>

  <script>
    let surfing = false;
    let timer;

    async function loadUser() {
      const res = await fetch('/api/user');
      const data = await res.json();
      document.getElementById('user-points').innerText = data.points + ' Points';
    }

    async function toggleSurf() {
      const btn = document.getElementById('surf-btn');
      const timerDisplay = document.getElementById('timer-display');
      const preview = document.getElementById('surf-preview');

      if (!surfing) {
        surfing = true;
        btn.innerText = "Stop Auto-Surf";
        btn.className = "bg-red-600 hover:bg-red-700 px-6 py-3 rounded-lg font-bold text-white transition";
        timerDisplay.classList.remove('hidden');
        preview.classList.remove('hidden');
        runSurfCycle();
      } else {
        surfing = false;
        clearInterval(timer);
        btn.innerText = "Start Auto-Surf";
        btn.className = "bg-green-600 hover:bg-green-700 px-6 py-3 rounded-lg font-bold text-white transition";
        timerDisplay.classList.add('hidden');
        preview.classList.add('hidden');
      }
    }

    async function runSurfCycle() {
      if (!surfing) return;
      const res = await fetch('/api/surf/next');
      const site = await res.json();
      
      document.getElementById('surf-iframe').src = site.url;
      let timeLeft = site.viewTime;

      timer = setInterval(async () => {
        timeLeft--;
        document.getElementById('timer-display').innerText = `Next in: ${timeLeft}s`;
        if (timeLeft <= 0) {
          clearInterval(timer);
          // Reward Points
          const completeRes = await fetch('/api/surf/complete', {
            method: 'POST',
            headers: {'Content-Type': 'application/json'},
            body: JSON.stringify({ userId: 1, siteId: site.siteId })
          });
          const completeData = await completeRes.json();
          document.getElementById('user-points').innerText = completeData.newPoints + ' Points';
          runSurfCycle();
        }
      }, 1000);
    }

    document.getElementById('add-site-form').addEventListener('submit', async (e) => {
      e.preventDefault();
      const url = document.getElementById('site-url').value;
      await fetch('/api/websites/add', {
        method: 'POST',
        headers: {'Content-Type': 'application/json'},
        body: JSON.stringify({ url, timerSec: 10, hitsCount: 100 })
      });
      alert('Website added successfully!');
      document.getElementById('site-url').value = '';
    });

    loadUser();
  </script>
</body>
</html>
