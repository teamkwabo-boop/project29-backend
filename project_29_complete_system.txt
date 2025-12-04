# Project 29 Complete System (Backend + Dashboard + Integrated Form)
# Directory-style representation with all final production-ready code

==============================
FILE: server.js
==============================

const express = require("express");
const sqlite3 = require("sqlite3").verbose();
const path = require("path");
const bodyParser = require("body-parser");
const bcrypt = require("bcrypt");
const jwt = require("jsonwebtoken");
const { Parser } = require("json2csv");

const app = express();
app.use(bodyParser.json());
app.use(express.urlencoded({ extended: true }));
app.use(express.static("public"));

const JWT_SECRET = process.env.JWT_SECRET || "CHANGE_THIS_SECRET";

// =============== DATABASE ==================
const db = new sqlite3.Database("project29.db");

db.serialize(() => {
  db.run(`CREATE TABLE IF NOT EXISTS supporters (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      name TEXT NOT NULL,
      dob TEXT NOT NULL,
      sex TEXT NOT NULL,
      location TEXT NOT NULL,
      community TEXT NOT NULL,
      clan TEXT NOT NULL,
      district TEXT NOT NULL,
      contact TEXT NOT NULL,
      email TEXT,
      currentAge INTEGER,
      age2029 INTEGER,
      UNIQUE(name, dob, contact)
  );`);

  db.run(`CREATE TABLE IF NOT EXISTS admins (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT UNIQUE,
    passwordHash TEXT
  );`);

  const username = "admin";
  const password = "changeMe123";
  bcrypt.hash(password, 10, (err, hash) => {
    db.run("INSERT OR IGNORE INTO admins(username, passwordHash) VALUES(?, ?)", [username, hash]);
  });
});

// =============== AUTH MIDDLEWARE ==================
function requireAuth(req, res, next) {
  const auth = req.headers.authorization;
  if (!auth) return res.status(401).json({ error: "Unauthorized" });

  const token = auth.split(" ")[1];
  try {
    req.user = jwt.verify(token, JWT_SECRET);
    next();
  } catch (e) {
    res.status(401).json({ error: "Invalid token" });
  }
}

// =============== LOGIN ==================
app.post("/api/admin/login", (req, res) => {
  const { username, password } = req.body;

  db.get("SELECT * FROM admins WHERE username = ?", [username], async (err, user) => {
    if (!user) return res.status(401).json({ error: "Invalid credentials" });
    const ok = await bcrypt.compare(password, user.passwordHash);
    if (!ok) return res.status(401).json({ error: "Invalid credentials" });

    const token = jwt.sign({ id: user.id, username: user.username }, JWT_SECRET, { expiresIn: "8h" });
    res.json({ token });
  });
});

// =============== SUPPORTER SUBMISSION ==================
app.post("/api/supporters", (req, res) => {
  const d = req.body;

  const stmt = db.prepare(`INSERT INTO supporters (name, dob, sex, location, community, clan, district, contact, email, currentAge, age2029)
    VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`);

  stmt.run(
    d.name,
    d.dob,
    d.sex,
    d.location,
    d.community,
    d.clan,
    d.district,
    d.contact,
    d.email,
    d.currentAge,
    d.age2029,
    err => {
      if (err) {
        if (err.message.includes("UNIQUE")) return res.status(400).json({ error: "Duplicate entry" });
        return res.status(500).json({ error: "Database error" });
      }
      res.json({ message: "Saved" });
    }
  );
});

// =============== LIST WITH FILTERS ==================
app.get("/api/supporters", (req, res) => {
  const { district, sex, q } = req.query;
  let sql = "SELECT * FROM supporters WHERE 1=1";
  const params = [];

  if (district) { sql += " AND district = ?"; params.push(district); }
  if (sex) { sql += " AND sex = ?"; params.push(sex); }
  if (q) {
    sql += " AND (name LIKE ? OR contact LIKE ? OR community LIKE ?)";
    params.push(`%${q}%`, `%${q}%`, `%${q}%`);
  }

  db.all(sql, params, (err, rows) => {
    if (err) return res.status(500).json({ error: "DB error" });
    res.json(rows);
  });
});

// =============== DASHBOARD STATS ==================
app.get("/api/stats", requireAuth, (req, res) => {
  db.serialize(() => {
    db.get("SELECT COUNT(*) AS total FROM supporters", (err, totalRow) => {
      db.all("SELECT sex, COUNT(*) AS count FROM supporters GROUP BY sex", (err, genderRows) => {
        db.all("SELECT district, COUNT(*) AS count FROM supporters GROUP BY district", (err, districtRows) => {
          res.json({
            totalSupporters: totalRow.total,
            genderBreakdown: genderRows,
            districtBreakdown: districtRows
          });
        });
      });
    });
  });
});

// =============== CSV EXPORT ==================
app.get("/api/export/csv", requireAuth, (req, res) => {
  db.all("SELECT * FROM supporters", (err, rows) => {
    const parser = new Parser();
    const csv = parser.parse(rows);
    res.header("Content-Type", "text/csv");
    res.attachment("supporters.csv");
    res.send(csv);
  });
});

// =============== SERVE DASHBOARD ==================
app.get("/dashboard", (req, res) => {
  res.sendFile(path.join(__dirname, "public/dashboard.html"));
});

// =============== SERVER START ==================
const PORT = process.env.PORT || 4000;
app.listen(PORT, () => console.log(`Running on port ${PORT}`));


==============================
FILE: public/form.html
==============================

<!DOCTYPE html>
<html>
<head>
  <title>Project 29 Supporter Form</title>
  <script>
    function calculate() {
      const dob = document.getElementById('dob').value;
      if (!dob) return;
      const birth = new Date(dob);
      const now = new Date();

      let age = now.getFullYear() - birth.getFullYear();
      if (now < new Date(now.getFullYear(), birth.getMonth(), birth.getDate())) age--;
      document.getElementById('currentAge').value = age;

      const target = new Date('2029-10-01');
      let fAge = target.getFullYear() - birth.getFullYear();
      if (target < new Date(target.getFullYear(), birth.getMonth(), birth.getDate())) fAge--;
      document.getElementById('age2029').value = fAge;
    }

    async function submitForm(e) {
      e.preventDefault();
      const form = new FormData(e.target);
      const payload = Object.fromEntries(form.entries());

      const res = await fetch('/api/supporters', {
        method: 'POST', headers: {'Content-Type': 'application/json'}, body: JSON.stringify(payload)
      });

      const msg = document.getElementById('msg');
      if (res.status === 400) return msg.textContent = "Duplicate supporter";
      if (!res.ok) return msg.textContent = "Server error";

      msg.textContent = "Saved";
      e.target.reset();
    }
  </script>
</head>
<body>
  <h2>Project 29 Supporter Form</h2>
  <form onsubmit="submitForm(event)">
    <label>Name *</label><input name="name" required>
    <label>Date of Birth *</label><input type="date" name="dob" id="dob" onchange="calculate()" required>
    <label>Current Age</label><input name="currentAge" id="currentAge" readonly>
    <label>Age in 2029</label><input name="age2029" id="age2029" readonly>
    <label>Sex *</label>
    <select name="sex" required><option>Male</option><option>Female</option></select>
    <label>Location *</label><input name="location" required>
    <label>Community *</label><input name="community" required>
    <label>Clan *</label><input name="clan" required>
    <label>Administrative District *</label><input name="district" required>
    <label>Contact *</label><input name="contact" required>
    <label>Email</label><input name="email">
    <button>Submit</button>
  </form>
  <p id="msg"></p>
</body>
</html>


==============================
FILE: public/dashboard.html
==============================

<!DOCTYPE html>
<html>
<head>
<title>Project 29 Dashboard</title>
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<script>
let TOKEN = null;

async function login() {
  const user = prompt("Admin Username:");
  const pass = prompt("Password:");
  
  const res = await fetch('/api/admin/login', {
    method: 'POST', headers: {'Content-Type':'application/json'}, body: JSON.stringify({username:user, password:pass})
  });

  if (!res.ok) return alert("Invalid");
  const data = await res.json();
  TOKEN = data.token;
  loadStats();
}

async function loadStats() {
  const res = await fetch('/api/stats', {
    headers: {Authorization: 'Bearer '+TOKEN}
  });
  const data = await res.json();

  document.getElementById('total').textContent = data.totalSupporters;

  const sex = data.genderBreakdown.map(x=>x.sex);
  const sexCount = data.genderBreakdown.map(x=>x.count);
  new Chart(document.getElementById('gender'), { type:'pie', data:{ labels:sex, datasets:[{data:sexCount}] } });

  const d = data.districtBreakdown.map(x=>x.district);
  const dCount = data.districtBreakdown.map(x=>x.count);
  new Chart(document.getElementById('district'), { type:'bar', data:{ labels:d, datasets:[{data:dCount}] } });
}
</script>
</head>
<body onload="login()">
<h2>Project 29 Dashboard</h2>
<p>Total Supporters: <span id="total"></span></p>
<canvas id="gender" width="300" height="300"></canvas>
<canvas id="district" width="300" height="300"></canvas>
</body>
</html>

