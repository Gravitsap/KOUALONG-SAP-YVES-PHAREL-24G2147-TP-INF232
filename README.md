# KOUALONG-SAP-YVES-PHAREL-24G2147-TP-INF232
this repository has been created to submit my assignment 
# 🌾 AgroSense Cameroun
Application de collecte et d'analyse des données agricoles — TP INF232 EC2

## Stack technique
- **Backend** : Python + Flask
- **Base de données** : SQLite (fichier local `agrosense.db`)
- **Frontend** : Jinja2 + HTML/CSS/JS + Chart.js

## Structure du projet
```
agrosense/
├── app.py                  ← Backend Flask (routes + BDD + stats)
├── agrosense.db            ← Base SQLite (créée automatiquement)
├── requirements.txt
├── templates/
│   ├── base.html           ← Layout commun (navbar, footer)
│   ├── index.html          ← Page d'accueil
│   ├── sondage.html        ← Formulaire de collecte
│   ├── success.html        ← Page de confirmation
│   └── dashboard.html      ← Tableau de bord + graphiques
└── static/
    ├── css/
    └── js/
```

## Lancement en local

```bash
# 1. Installer les dépendances
pip install -r requirements.txt

# 2. Lancer l'application
python app.py

# 3. Ouvrir dans le navigateur
# http://127.0.0.1:5000
```

## Routes disponibles
| URL | Description |
|-----|-------------|
| `GET /` | Redirection vers introduction |
| `GET /intro` | Page d'introduction |
| `GET /collect` | Formulaire de collecte de données |
| `POST /collect` | Soumission des données |
| `GET /success` | Confirmation soumission |
| `GET /publish` | Publication des résultats (dashboard) |
| `GET /api/stats` | API JSON — statistiques |
| `GET /api/reponses` | API JSON — liste réponses |

## Déploiement en ligne

### PythonAnywhere (recommandé)
1. Créer un compte sur https://www.pythonanywhere.com
2. Créer une nouvelle web app Flask
3. Uploader les fichiers du projet
4. Configurer le WSGI pour pointer vers `wsgi.py`
5. L'application sera accessible via votre domaine PythonAnywhere

### Render.com (gratuit)
1. Créer un compte sur https://render.com
2. New → Web Service → connecter votre repo GitHub
3. Build command : `pip install -r requirements.txt`
4. Start command : `python app.py`
5. Copier l'URL générée et l'envoyer au professeur

code en python
"""
AgroSense Cameroun — Application Flask
Backend : Flask + SQLite  |  Frontend : Jinja2
TP INF232 EC2
"""
from flask import Flask, render_template, request, redirect, url_for, jsonify, flash, session
import sqlite3, json, os
from datetime import datetime
from statistics import mean, median, stdev
from collections import Counter

app = Flask(__name__)
app.secret_key = "agrosense_inf232_2026_secret"
DB  = os.path.join(os.path.dirname(__file__), "agrosense.db")

# ══════════════════════════════════════════════════════════════════════════════
# BASE DE DONNÉES
# ══════════════════════════════════════════════════════════════════════════════
def get_db():
    conn = sqlite3.connect(DB)
    conn.row_factory = sqlite3.Row
    return conn

def init_db():
    with get_db() as c:
        c.executescript("""
        CREATE TABLE IF NOT EXISTS reponses (
            id              INTEGER PRIMARY KEY AUTOINCREMENT,
            created_at      TEXT    NOT NULL,
            region          TEXT    NOT NULL,
            departement     TEXT,
            statut          TEXT    NOT NULL,
            age             TEXT    NOT NULL,
            genre           TEXT    NOT NULL,
            cultures        TEXT    NOT NULL,
            surface         REAL    NOT NULL,
            mode_culture    TEXT    NOT NULL,
            rendement       REAL    NOT NULL,
            saison          TEXT    NOT NULL,
            annee           INTEGER NOT NULL,
            defis           TEXT,
            prix_vente      REAL,
            canal_distrib   TEXT,
            distance_marche TEXT,
            commentaire     TEXT
        );
        """)

# ══════════════════════════════════════════════════════════════════════════════
# STATISTIQUES
# ══════════════════════════════════════════════════════════════════════════════
def compute_stats(rows):
    if not rows:
        return None

    rendements = [r["rendement"] for r in rows if r["rendement"] is not None]
    surfaces   = [r["surface"]   for r in rows if r["surface"]   is not None]
    prix_list  = [r["prix_vente"] for r in rows if r["prix_vente"] is not None]

    all_cultures, all_defis = [], []
    for r in rows:
        try: all_cultures.extend(json.loads(r["cultures"] or "[]"))
        except: pass
        try: all_defis.extend(json.loads(r["defis"] or "[]"))
        except: pass

    def sm(lst): return round(mean(lst), 1)   if lst else 0
    def sd(lst): return round(median(lst), 1) if lst else 0
    def sv(lst): return round(stdev(lst), 1)  if len(lst) >= 2 else 0

    culture_counts = dict(Counter(all_cultures).most_common(8))
    defi_counts    = dict(Counter(all_defis).most_common(8))
    region_counts  = dict(Counter(r["region"] for r in rows).most_common(10))
    mode_counts    = dict(Counter(r["mode_culture"] for r in rows))
    saison_counts  = dict(Counter(r["saison"] for r in rows))
    statut_counts  = dict(Counter(r["statut"] for r in rows))
    scatter        = [{"x": r["surface"], "y": r["rendement"]}
                      for r in rows if r["surface"] and r["rendement"]]

    moy_r = sm(rendements)
    moy_s = sm(surfaces)
    moy_p = sm(prix_list)

    insights = []
    if moy_r and moy_r < 800:
        insights.append({"icon":"⚠️","type":"warn","title":"Rendement faible",
            "text":f"Rendement moyen {moy_r} kg/ha — sous le seuil FAO (800 kg/ha). Un appui technique est recommandé."})
    elif moy_r:
        insights.append({"icon":"✅","type":"good","title":"Rendement satisfaisant",
            "text":f"Rendement moyen {moy_r} kg/ha — au-dessus du seuil FAO. Bonne productivité globale."})

    if culture_counts:
        top = list(culture_counts.keys())[0]
        insights.append({"icon":"🌾","type":"info","title":"Culture dominante",
            "text":f"Le {top} domine avec {culture_counts[top]} déclarations — axe stratégique identifié."})

    if defi_counts:
        top_d = list(defi_counts.keys())[0]
        insights.append({"icon":"🚨","type":"warn","title":"Défi prioritaire",
            "text":f"Le défi « {top_d} » est le plus fréquent ({defi_counts[top_d]} mentions). Intervention prioritaire conseillée."})

    if moy_p and moy_p < 200:
        insights.append({"icon":"📉","type":"warn","title":"Prix de vente bas",
            "text":f"Prix moyen {moy_p} F CFA/kg — sous le seuil de rentabilité. Accès aux marchés à améliorer."})

    return {
        "total":          len(rows),
        "moy_rendement":  moy_r,
        "med_rendement":  sd(rendements),
        "std_rendement":  sv(rendements),
        "moy_surface":    moy_s,
        "moy_prix":       moy_p,
        "culture_counts": culture_counts,
        "defi_counts":    defi_counts,
        "region_counts":  region_counts,
        "mode_counts":    mode_counts,
        "saison_counts":  saison_counts,
        "statut_counts":  statut_counts,
        "scatter":        scatter,
        "insights":       insights,
        "top_culture":    list(culture_counts.keys())[0] if culture_counts else "—",
        "top_region":     list(region_counts.keys())[0]  if region_counts  else "—",
    }

# ══════════════════════════════════════════════════════════════════════════════
# ROUTES
# ══════════════════════════════════════════════════════════════════════════════
@app.route("/")
def home():
    return redirect(url_for("index"))

@app.route("/intro")
def index():
    with get_db() as c:
        total   = c.execute("SELECT COUNT(*) FROM reponses").fetchone()[0]
        regions = c.execute("SELECT COUNT(DISTINCT region) FROM reponses").fetchone()[0]
    return render_template("index.html", total=total, regions=regions)

@app.route("/collect", methods=["GET"])
def sondage():
    return render_template("sondage.html")

@app.route("/collect", methods=["POST"])
def sondage_submit():
    f        = request.form
    cultures = request.form.getlist("cultures")
    defis    = request.form.getlist("defis")
    required = ["region","statut","age","genre","surface","rendement","saison","annee","mode_culture"]

    if any(not f.get(k,"").strip() for k in required) or not cultures:
        flash("Veuillez remplir tous les champs obligatoires (*).", "error")
        return redirect(url_for("sondage"))

    try:
        surface   = float(f["surface"])
        rendement = float(f["rendement"])
        annee     = int(f["annee"])
        prix      = float(f["prix_vente"]) if f.get("prix_vente") else None
    except ValueError:
        flash("Valeurs numériques invalides.", "error")
        return redirect(url_for("sondage"))

    with get_db() as c:
        c.execute("""INSERT INTO reponses
            (created_at,region,departement,statut,age,genre,
             cultures,surface,mode_culture,rendement,saison,annee,
             defis,prix_vente,canal_distrib,distance_marche,commentaire)
            VALUES(?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?,?)""", (
            datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
            f["region"], f.get("departement",""),
            f["statut"], f["age"], f["genre"],
            json.dumps(cultures, ensure_ascii=False),
            surface, f["mode_culture"], rendement,
            f["saison"], annee,
            json.dumps(defis, ensure_ascii=False) if defis else None,
            prix, f.get("canal_distrib",""),
            f.get("distance_marche",""), f.get("commentaire","")
        ))

    return redirect(url_for("success"))

@app.route("/success")
def success():
    with get_db() as c:
        total = c.execute("SELECT COUNT(*) FROM reponses").fetchone()[0]
    return render_template("success.html", total=total)

@app.route("/publish")
def dashboard():
    with get_db() as c:
        rows = [dict(r) for r in c.execute(
            "SELECT * FROM reponses ORDER BY created_at DESC").fetchall()]
    stats = compute_stats(rows)
    return render_template("dashboard.html", stats=stats)

@app.route("/api/stats")
def api_stats():
    with get_db() as c:
        rows = [dict(r) for r in c.execute("SELECT * FROM reponses").fetchall()]
    return jsonify(compute_stats(rows) or {"total": 0})

@app.route("/api/reponses")
def api_reponses():
    with get_db() as c:
        rows = [dict(r) for r in c.execute(
            "SELECT * FROM reponses ORDER BY created_at DESC LIMIT 200").fetchall()]
    return jsonify(rows)

# ══════════════════════════════════════════════════════════════════════════════
if __name__ == "__main__":
    init_db()
    print("🌾 AgroSense — http://127.0.0.1:5000")
    app.run(debug=True)

code html 

<!DOCTYPE html>
<html lang="fr">
<head>
<meta charset="UTF-8"/>
<meta name="viewport" content="width=device-width, initial-scale=1.0"/>
<title>{% block title %}AgroSense Cameroun{% endblock %}</title>
<link href="https://fonts.googleapis.com/css2?family=Playfair+Display:wght@400;700;900&family=DM+Sans:wght@300;400;500;600&display=swap" rel="stylesheet"/>
<script src="https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.1/chart.umd.min.js"></script>
<style>
/* ── RESET & BASE ─────────────────────────────────────────────────────── */
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
html { scroll-behavior: smooth; }
:root {
  --bg:      #0d1a0f;
  --surf:    #132016;
  --surf2:   #1a2d1e;
  --surf3:   #223528;
  --border:  rgba(255,255,255,0.07);
  --green:   #4ade80;
  --green2:  #22c55e;
  --orange:  #fb923c;
  --gold:    #fbbf24;
  --text:    #e2f0e6;
  --muted:   #6b7c70;
  --r:       14px;
}
body {
  background: var(--bg);
  color: var(--text);
  font-family: 'DM Sans', sans-serif;
  min-height: 100vh;
}
body::before {
  content: '';
  position: fixed; inset: 0; z-index: 0; pointer-events: none;
  background:
    radial-gradient(ellipse 60% 45% at 10% 15%, rgba(74,222,128,.06) 0%, transparent 65%),
    radial-gradient(ellipse 50% 40% at 90% 85%, rgba(251,146,60,.05) 0%, transparent 65%),
    radial-gradient(ellipse 40% 30% at 55% 50%, rgba(251,191,36,.03) 0%, transparent 60%);
}

/* ── NAV ──────────────────────────────────────────────────────────────── */
.navbar {
  position: fixed; top: 0; left: 0; right: 0; z-index: 200;
  background: rgba(13,26,15,.92); backdrop-filter: blur(20px);
  border-bottom: 1px solid var(--border);
  padding: 0 24px; height: 62px;
  display: flex; align-items: center; justify-content: space-between;
}
.nav-brand {
  font-family: 'Playfair Display', serif;
  font-size: 22px; font-weight: 900; color: var(--green);
  text-decoration: none; letter-spacing: -0.5px;
}
.nav-brand span { color: var(--text); }
.nav-links { display: flex; gap: 4px; }
.nav-link {
  padding: 8px 18px; border-radius: 50px;
  color: var(--muted); text-decoration: none;
  font-size: 14px; font-weight: 500;
  transition: all .2s;
}
.nav-link:hover { color: var(--text); background: var(--surf2); }
.nav-link.active { background: var(--surf3); color: var(--green); font-weight: 600; }
.nav-badge {
  font-size: 11px; background: rgba(74,222,128,.12);
  border: 1px solid rgba(74,222,128,.25);
  color: var(--green); padding: 4px 12px; border-radius: 50px; font-weight: 600;
}

/* ── FLASH MESSAGES ───────────────────────────────────────────────────── */
.flash-wrap { position: fixed; top: 74px; right: 20px; z-index: 300; display: flex; flex-direction: column; gap: 8px; }
.flash {
  padding: 12px 20px; border-radius: 10px;
  font-size: 14px; font-weight: 500; max-width: 380px;
  animation: slideIn .3s ease;
}
.flash.success { background: rgba(74,222,128,.15); border: 1px solid rgba(74,222,128,.3); color: var(--green); }
.flash.error   { background: rgba(248,113,113,.15); border: 1px solid rgba(248,113,113,.3); color: #f87171; }
@keyframes slideIn { from{opacity:0;transform:translateX(20px)} to{opacity:1;transform:translateX(0)} }

/* ── PAGE WRAPPER ─────────────────────────────────────────────────────── */
.page { position: relative; z-index: 1; padding-top: 62px; }

/* ── BUTTONS ──────────────────────────────────────────────────────────── */
.btn-primary {
  display: inline-block; padding: 14px 32px;
  background: var(--green); color: #0d1a0f;
  border: none; border-radius: 50px;
  font-family: 'DM Sans', sans-serif; font-size: 14px; font-weight: 700;
  cursor: pointer; text-decoration: none; transition: all .2s;
}
.btn-primary:hover { transform: translateY(-2px); box-shadow: 0 12px 30px rgba(74,222,128,.28); }
.btn-secondary {
  display: inline-block; padding: 14px 32px;
  background: transparent; color: var(--text);
  border: 1px solid var(--border); border-radius: 50px;
  font-family: 'DM Sans', sans-serif; font-size: 14px; font-weight: 500;
  cursor: pointer; text-decoration: none; transition: all .2s;
}
.btn-secondary:hover { border-color: var(--green); color: var(--green); }

/* ── CARDS ────────────────────────────────────────────────────────────── */
.card {
  background: var(--surf); border: 1px solid var(--border);
  border-radius: var(--r); transition: border-color .2s;
}
.card:hover { border-color: rgba(74,222,128,.18); }

/* ── FOOTER ───────────────────────────────────────────────────────────── */
footer {
  text-align: center; padding: 32px 24px;
  border-top: 1px solid var(--border);
  color: var(--muted); font-size: 13px;
  position: relative; z-index: 1;
  margin-top: 60px;
}
footer span { color: var(--green); }

/* ── ANIMATIONS ───────────────────────────────────────────────────────── */
@keyframes fadeUp { from{opacity:0;transform:translateY(18px)} to{opacity:1;transform:translateY(0)} }
.fade-up { animation: fadeUp .5s ease both; }
.fade-up-2 { animation: fadeUp .5s .1s ease both; }
.fade-up-3 { animation: fadeUp .5s .2s ease both; }
.fade-up-4 { animation: fadeUp .5s .3s ease both; }

@media(max-width:640px){
  .nav-links { display: none; }
  .nav-badge { display: none; }
}
</style>
{% block extra_css %}{% endblock %}
</head>
<body>

<!-- NAVBAR -->
<nav class="navbar">
  <a href="{{ url_for('index') }}" class="nav-brand">🌾 Agro<span>Sense</span></a>
  <div class="nav-links">
    <a href="{{ url_for('index') }}"     class="nav-link {% if request.endpoint=='index' %}active{% endif %}">Introduction</a>
    <a href="{{ url_for('sondage') }}"   class="nav-link {% if request.endpoint in ['sondage','sondage_submit'] %}active{% endif %}">Collecte de données</a>
    <a href="{{ url_for('dashboard') }}" class="nav-link {% if request.endpoint=='dashboard' %}active{% endif %}">Publication des résultats</a>
  </div>
  <div class="nav-badge">INF232 — TP EC2</div>
</nav>

<!-- FLASH -->
{% with messages = get_flashed_messages(with_categories=true) %}
  {% if messages %}
    <div class="flash-wrap">
      {% for cat, msg in messages %}
        <div class="flash {{ cat }}">{{ msg }}</div>
      {% endfor %}
    </div>
  {% endif %}
{% endwith %}

<!-- PAGE -->
<div class="page">
  {% block content %}{% endblock %}
</div>

<footer>
  <strong><span>AgroSense Cameroun</span></strong> — Collecte & Analyse de données agricoles<br/>
  TP INF232 EC2 · Flask + SQLite + Jinja2 · Avril 2026
</footer>

{% block extra_js %}{% endblock %}
</body>
</html>


{% extends "base.html" %}
{% block title %}Dashboard — AgroSense Cameroun{% endblock %}

{% block extra_css %}
<style>
.dash-wrap { max-width: 1100px; margin: 0 auto; padding: 48px 24px 80px; }

/* HEADER */
.dash-hd { display:flex; align-items:flex-start; justify-content:space-between; margin-bottom:36px; flex-wrap:wrap; gap:14px; }
.dash-hd h2 { font-family:'Playfair Display',serif; font-size:40px; font-weight:700; }
.dash-hd p  { color:var(--muted); font-size:13px; margin-top:5px; }
.live-pill {
  display:flex; align-items:center; gap:7px;
  background:rgba(74,222,128,.07); border:1px solid rgba(74,222,128,.18);
  border-radius:50px; padding:7px 16px; font-size:12px; color:var(--green); font-weight:600;
}
.live-dot { width:7px; height:7px; border-radius:50%; background:var(--green); animation:blink 1.4s infinite; }
@keyframes blink { 0%,100%{opacity:1} 50%{opacity:.25} }

/* EMPTY */
.empty-state { text-align:center; padding:100px 40px; }
.empty-state .e-ico { font-size:60px; margin-bottom:18px; opacity:.4; }
.empty-state h3 { font-family:'Playfair Display',serif; font-size:28px; margin-bottom:10px; }
.empty-state p  { color:var(--muted); max-width:380px; margin:0 auto 28px; line-height:1.65; }

/* KPIs */
.kpi-grid { display:grid; grid-template-columns:repeat(auto-fit,minmax(200px,1fr)); gap:14px; margin-bottom:28px; }
.kpi-card {
  background:var(--surf); border:1px solid var(--border);
  border-radius:var(--r); padding:22px; position:relative; overflow:hidden;
  transition:transform .2s, border-color .2s;
}
.kpi-card:hover { transform:translateY(-3px); border-color:rgba(74,222,128,.2); }
.kpi-bar { position:absolute; top:0; left:0; right:0; height:2px; }
.kpi-lbl { font-size:10px; color:var(--muted); text-transform:uppercase; letter-spacing:1px; margin-bottom:10px; font-weight:600; }
.kpi-val { font-family:'Playfair Display',serif; font-size:44px; font-weight:900; line-height:1; margin-bottom:5px; }
.kpi-sub { font-size:11px; color:var(--muted); }

/* CHARTS */
.chart-grid { display:grid; grid-template-columns:repeat(auto-fit,minmax(320px,1fr)); gap:16px; margin-bottom:16px; }
.chart-card {
  background:var(--surf); border:1px solid var(--border);
  border-radius:var(--r); padding:24px;
  transition:border-color .2s;
}
.chart-card:hover { border-color:rgba(74,222,128,.15); }
.chart-card.wide { grid-column:1/-1; }
.chart-ttl { font-size:14px; font-weight:700; margin-bottom:4px; }
.chart-sub { font-size:11px; color:var(--muted); margin-bottom:20px; }
.chart-box { position:relative; height:230px; }
.chart-box.tall { height:280px; }

/* INSIGHTS */
.insight-grid { display:grid; grid-template-columns:repeat(auto-fit,minmax(260px,1fr)); gap:14px; margin-top:16px; }
.insight-card {
  background:var(--surf); border:1px solid var(--border);
  border-radius:var(--r); padding:20px;
  display:flex; gap:14px; align-items:flex-start;
}
.ins-ico { width:42px; height:42px; border-radius:11px; display:flex; align-items:center; justify-content:center; font-size:18px; flex-shrink:0; }
.ins-ico.warn { background:rgba(251,146,60,.12); }
.ins-ico.good { background:rgba(74,222,128,.12); }
.ins-ico.info { background:rgba(147,197,253,.12); }
.ins-txt h4 { font-size:13px; font-weight:700; margin-bottom:5px; }
.ins-txt p  { font-size:12px; color:var(--muted); line-height:1.65; }

/* RECENT TABLE */
.recent-section { margin-top: 32px; }
.recent-section h3 { font-family:'Playfair Display',serif; font-size:24px; font-weight:700; margin-bottom:16px; }
.table-wrap { overflow-x:auto; }
table { width:100%; border-collapse:collapse; font-size:13px; }
thead { background:var(--surf2); }
th,td { padding:11px 14px; text-align:left; border-bottom:1px solid var(--border); }
th { color:var(--muted); font-weight:600; font-size:11px; text-transform:uppercase; letter-spacing:.5px; }
tr:hover { background:rgba(255,255,255,.02); }
.badge {
  display:inline-block; padding:3px 10px; border-radius:50px;
  font-size:11px; font-weight:600;
}
.badge-g { background:rgba(74,222,128,.12); color:var(--green); }
.badge-o { background:rgba(251,146,60,.12); color:var(--orange); }
</style>
{% endblock %}

{% block content %}
<div class="dash-wrap">

  <!-- HEADER -->
  <div class="dash-hd fade-up">
    <div>
      <h2>Tableau de bord</h2>
      <p>Analyse descriptive des données agricoles collectées</p>
    </div>
    <div class="live-pill"><div class="live-dot"></div>Données en direct</div>
  </div>

  {% if not stats %}
  <!-- EMPTY STATE -->
  <div class="empty-state">
    <div class="e-ico">📊</div>
    <h3>Aucune donnée disponible</h3>
    <p>Soyez le premier à remplir le sondage pour voir les statistiques apparaître ici.</p>
    <a href="{{ url_for('sondage') }}" class="btn-primary">🌱 Remplir le sondage</a>
  </div>

  {% else %}

  <!-- KPIs -->
  <div class="kpi-grid fade-up">
    <div class="kpi-card">
      <div class="kpi-bar" style="background:var(--green)"></div>
      <div class="kpi-lbl">Réponses totales</div>
      <div class="kpi-val" style="color:var(--green)">{{ stats.total }}</div>
      <div class="kpi-sub">agriculteurs</div>
    </div>
    <div class="kpi-card">
      <div class="kpi-bar" style="background:var(--orange)"></div>
      <div class="kpi-lbl">Rendement moyen</div>
      <div class="kpi-val" style="color:var(--orange)">{{ stats.moy_rendement }}</div>
      <div class="kpi-sub">kg / hectare</div>
    </div>
    <div class="kpi-card">
      <div class="kpi-bar" style="background:var(--gold)"></div>
      <div class="kpi-lbl">Surface moyenne</div>
      <div class="kpi-val" style="color:var(--gold)">{{ stats.moy_surface }}</div>
      <div class="kpi-sub">hectares</div>
    </div>
    <div class="kpi-card">
      <div class="kpi-bar" style="background:#a78bfa"></div>
      <div class="kpi-lbl">Culture dominante</div>
      <div class="kpi-val" style="color:#a78bfa;font-size:28px;">{{ stats.top_culture }}</div>
      <div class="kpi-sub">la plus déclarée</div>
    </div>
    <div class="kpi-card">
      <div class="kpi-bar" style="background:#60a5fa"></div>
      <div class="kpi-lbl">Prix moyen</div>
      <div class="kpi-val" style="color:#60a5fa">{{ stats.moy_prix }}</div>
      <div class="kpi-sub">F CFA / kg</div>
    </div>
  </div>

  <!-- CHARTS ROW 1 -->
  <div class="chart-grid">
    <div class="chart-card">
      <div class="chart-ttl">🌿 Cultures déclarées</div>
      <div class="chart-sub">Fréquence par type de culture</div>
      <div class="chart-box"><canvas id="chartCultures"></canvas></div>
    </div>
    <div class="chart-card">
      <div class="chart-ttl">🗺️ Répartition par région</div>
      <div class="chart-sub">Nombre de réponses par région</div>
      <div class="chart-box"><canvas id="chartRegions"></canvas></div>
    </div>
    <div class="chart-card">
      <div class="chart-ttl">🚜 Mode de culture</div>
      <div class="chart-sub">Répartition des méthodes agricoles</div>
      <div class="chart-box"><canvas id="chartModes"></canvas></div>
    </div>
  </div>

  <!-- CHARTS ROW 2 -->
  <div class="chart-grid">
    <div class="chart-card wide">
      <div class="chart-ttl">⚠️ Principaux défis agricoles</div>
      <div class="chart-sub">Nombre de déclarations par type de défi</div>
      <div class="chart-box tall"><canvas id="chartDefis"></canvas></div>
    </div>
  </div>

  <!-- CHARTS ROW 3 -->
  <div class="chart-grid">
    <div class="chart-card">
      <div class="chart-ttl">📅 Saisons de récolte</div>
      <div class="chart-sub">Distribution par saison</div>
      <div class="chart-box"><canvas id="chartSaisons"></canvas></div>
    </div>
    <div class="chart-card">
      <div class="chart-ttl">📈 Rendement vs Surface</div>
      <div class="chart-sub">Corrélation observée (scatter plot)</div>
      <div class="chart-box"><canvas id="chartScatter"></canvas></div>
    </div>
    <div class="chart-card">
      <div class="chart-ttl">👤 Statut des agriculteurs</div>
      <div class="chart-sub">Répartition par type d'exploitation</div>
      <div class="chart-box"><canvas id="chartStatuts"></canvas></div>
    </div>
  </div>

  <!-- INSIGHTS -->
  {% if stats.insights %}
  <div class="insight-grid">
    {% for ins in stats.insights %}
    <div class="insight-card">
      <div class="ins-ico {{ ins.type }}">{{ ins.icon }}</div>
      <div class="ins-txt">
        <h4>{{ ins.title }}</h4>
        <p>{{ ins.text }}</p>
      </div>
    </div>
    {% endfor %}
  </div>
  {% endif %}

  {% endif %}
</div>
{% endblock %}

{% block extra_js %}
{% if stats %}
<script>
// ── DONNÉES DEPUIS FLASK ──────────────────────────────────────────────────────
const STATS = {{ stats | tojson }};

const CHART_DEFAULTS = {
  responsive: true,
  maintainAspectRatio: false,
  plugins: { legend: { display: false } },
  scales: {
    x: { grid: { color: 'rgba(255,255,255,0.05)' }, ticks: { color: '#6b7c70', font: { family: 'DM Sans' } } },
    y: { grid: { color: 'rgba(255,255,255,0.05)' }, ticks: { color: '#6b7c70', font: { family: 'DM Sans' }, stepSize: 1 } }
  }
};

const COLORS = [
  'rgba(74,222,128,.75)',  'rgba(251,146,60,.75)', 'rgba(251,191,36,.75)',
  'rgba(167,139,250,.75)','rgba(96,165,250,.75)',  'rgba(249,115,22,.75)',
  'rgba(52,211,153,.75)', 'rgba(248,113,113,.75)', 'rgba(163,230,53,.75)',
  'rgba(129,140,248,.75)','rgba(232,121,249,.75)', 'rgba(45,212,191,.75)'
];

// ── 1. CULTURES ───────────────────────────────────────────────────────────────
new Chart(document.getElementById('chartCultures'), {
  type: 'bar',
  data: {
    labels: Object.keys(STATS.culture_counts),
    datasets: [{ data: Object.values(STATS.culture_counts), backgroundColor: COLORS, borderRadius: 6, borderWidth: 0 }]
  },
  options: { ...CHART_DEFAULTS }
});

// ── 2. RÉGIONS ────────────────────────────────────────────────────────────────
new Chart(document.getElementById('chartRegions'), {
  type: 'doughnut',
  data: {
    labels: Object.keys(STATS.region_counts),
    datasets: [{ data: Object.values(STATS.region_counts), backgroundColor: COLORS, borderWidth: 0, hoverOffset: 8 }]
  },
  options: {
    responsive: true, maintainAspectRatio: false, cutout: '62%',
    plugins: { legend: { position: 'bottom', labels: { color: '#6b7c70', font: { family: 'DM Sans' }, boxWidth: 12, padding: 10 } } }
  }
});

// ── 3. MODES DE CULTURE ───────────────────────────────────────────────────────
new Chart(document.getElementById('chartModes'), {
  type: 'doughnut',
  data: {
    labels: Object.keys(STATS.mode_counts),
    datasets: [{ data: Object.values(STATS.mode_counts), backgroundColor: COLORS.slice(4), borderWidth: 0, hoverOffset: 8 }]
  },
  options: {
    responsive: true, maintainAspectRatio: false, cutout: '62%',
    plugins: { legend: { position: 'bottom', labels: { color: '#6b7c70', font: { family: 'DM Sans' }, boxWidth: 12, padding: 10 } } }
  }
});

// ── 4. DÉFIS (BARRES HORIZONTALES) ────────────────────────────────────────────
new Chart(document.getElementById('chartDefis'), {
  type: 'bar',
  data: {
    labels: Object.keys(STATS.defi_counts),
    datasets: [{ data: Object.values(STATS.defi_counts), backgroundColor: COLORS.map((_,i) => COLORS[i % COLORS.length]), borderRadius: 6, borderWidth: 0 }]
  },
  options: {
    ...CHART_DEFAULTS,
    indexAxis: 'y',
    scales: {
      x: { grid: { color: 'rgba(255,255,255,0.05)' }, ticks: { color: '#6b7c70', stepSize: 1 } },
      y: { grid: { display: false }, ticks: { color: '#9ca3af' } }
    }
  }
});

// ── 5. SAISONS ────────────────────────────────────────────────────────────────
new Chart(document.getElementById('chartSaisons'), {
  type: 'bar',
  data: {
    labels: Object.keys(STATS.saison_counts),
    datasets: [{ data: Object.values(STATS.saison_counts), backgroundColor: 'rgba(251,191,36,.7)', borderRadius: 6, borderWidth: 0 }]
  },
  options: { ...CHART_DEFAULTS }
});

// ── 6. SCATTER : RENDEMENT VS SURFACE ────────────────────────────────────────
new Chart(document.getElementById('chartScatter'), {
  type: 'scatter',
  data: {
    datasets: [{
      data: STATS.scatter,
      backgroundColor: 'rgba(74,222,128,.55)',
      pointRadius: 6, pointHoverRadius: 9
    }]
  },
  options: {
    responsive: true, maintainAspectRatio: false,
    plugins: { legend: { display: false } },
    scales: {
      x: { title: { display: true, text: 'Surface (ha)', color: '#6b7c70' }, grid: { color: 'rgba(255,255,255,0.05)' }, ticks: { color: '#6b7c70' } },
      y: { title: { display: true, text: 'Rendement (kg/ha)', color: '#6b7c70' }, grid: { color: 'rgba(255,255,255,0.05)' }, ticks: { color: '#6b7c70' } }
    }
  }
});

// ── 7. STATUTS ────────────────────────────────────────────────────────────────
new Chart(document.getElementById('chartStatuts'), {
  type: 'doughnut',
  data: {
    labels: Object.keys(STATS.statut_counts),
    datasets: [{ data: Object.values(STATS.statut_counts), backgroundColor: COLORS.slice(2), borderWidth: 0, hoverOffset: 8 }]
  },
  options: {
    responsive: true, maintainAspectRatio: false, cutout: '62%',
    plugins: { legend: { position: 'bottom', labels: { color: '#6b7c70', font: { family: 'DM Sans' }, boxWidth: 12, padding: 8 } } }
  }
});
</script>
{% endif %}
{% endblock %}


{% extends "base.html" %}
{% block title %}AgroSense Cameroun — Accueil{% endblock %}

{% block extra_css %}
<style>
.hero {
  min-height: calc(100vh - 62px);
  display: flex; flex-direction: column;
  align-items: center; justify-content: center;
  text-align: center; padding: 80px 24px 60px;
}
.hero-badge {
  display: inline-flex; align-items: center; gap: 8px;
  background: rgba(74,222,128,.08); border: 1px solid rgba(74,222,128,.22);
  border-radius: 50px; padding: 7px 18px; font-size: 12px;
  color: var(--green); font-weight: 600; letter-spacing: .4px;
  margin-bottom: 32px;
}
.hero h1 {
  font-family: 'Playfair Display', serif;
  font-size: clamp(52px,9vw,108px); font-weight: 900;
  line-height: .95; margin-bottom: 24px; letter-spacing: -2px;
}
.hero h1 em { font-style: italic; color: var(--green); }
.hero-desc {
  font-size: 17px; color: var(--muted); max-width: 500px;
  line-height: 1.75; margin-bottom: 44px;
}
.hero-btns { display: flex; gap: 12px; flex-wrap: wrap; justify-content: center; }

.stats-strip {
  display: flex; gap: 52px; margin-top: 72px;
  flex-wrap: wrap; justify-content: center;
}
.stat-item { text-align: center; }
.stat-num {
  font-family: 'Playfair Display', serif;
  font-size: 46px; font-weight: 900; color: var(--green); line-height: 1;
}
.stat-label { font-size: 11px; color: var(--muted); margin-top: 6px; text-transform: uppercase; letter-spacing: .6px; }

/* Features */
.features { padding: 80px 24px; max-width: 1060px; margin: 0 auto; }
.features-title {
  font-family: 'Playfair Display', serif;
  font-size: 36px; font-weight: 700; text-align: center; margin-bottom: 48px;
}
.features-grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(280px,1fr)); gap: 20px; }
.feature-card {
  background: var(--surf); border: 1px solid var(--border);
  border-radius: var(--r); padding: 28px;
  transition: transform .2s, border-color .2s;
}
.feature-card:hover { transform: translateY(-4px); border-color: rgba(74,222,128,.2); }
.feature-icon {
  width: 48px; height: 48px; border-radius: 12px;
  display: flex; align-items: center; justify-content: center;
  font-size: 22px; margin-bottom: 16px;
}
.fi-g { background: rgba(74,222,128,.1); }
.fi-o { background: rgba(251,146,60,.1); }
.fi-y { background: rgba(251,191,36,.1); }
.fi-b { background: rgba(147,197,253,.1); }
.feature-card h3 { font-size: 16px; font-weight: 700; margin-bottom: 10px; }
.feature-card p  { font-size: 13px; color: var(--muted); line-height: 1.65; }

/* How it works */
.how { padding: 60px 24px; max-width: 900px; margin: 0 auto; }
.how-title {
  font-family: 'Playfair Display', serif;
  font-size: 32px; font-weight: 700; text-align: center; margin-bottom: 44px;
}
.steps { display: grid; grid-template-columns: repeat(auto-fit, minmax(220px,1fr)); gap: 16px; }
.step {
  background: var(--surf); border: 1px solid var(--border);
  border-radius: var(--r); padding: 24px; text-align: center;
}
.step-num {
  font-family: 'Playfair Display', serif;
  font-size: 42px; font-weight: 900; color: var(--green);
  margin-bottom: 12px; line-height: 1;
}
.step h4 { font-size: 15px; font-weight: 700; margin-bottom: 8px; }
.step p  { font-size: 13px; color: var(--muted); line-height: 1.6; }

/* CTA */
.cta {
  text-align: center; padding: 60px 24px 40px;
  border-top: 1px solid var(--border); margin-top: 40px;
}
.cta h2 {
  font-family: 'Playfair Display', serif;
  font-size: 32px; font-weight: 700; margin-bottom: 14px;
}
.cta p { color: var(--muted); margin-bottom: 28px; font-size: 15px; }
</style>
{% endblock %}

{% block content %}

<!-- HERO -->
<section class="hero">
  <div class="hero-badge fade-up">🌍 Projet INF232 — Collecte &amp; Analyse de données agricoles</div>
  <h1 class="fade-up-2">Agro<em>Sense</em><br/>Cameroun</h1>
  <p class="hero-desc fade-up-3">
    Plateforme de collecte citoyenne et d'analyse des données agricoles au Cameroun.
    Contribuez à la recherche et visualisez les résultats en temps réel.
  </p>
  <div class="hero-btns fade-up-4">
    <a href="{{ url_for('sondage') }}" class="btn-primary">🌱 Collecte de données</a>
    <a href="{{ url_for('dashboard') }}" class="btn-secondary">📊 Voir les résultats</a>
  </div>

  <div class="stats-strip fade-up-4">
    <div class="stat-item">
      <div class="stat-num">{{ total }}</div>
      <div class="stat-label">Réponses collectées</div>
    </div>
    <div class="stat-item">
      <div class="stat-num">{{ regions }}</div>
      <div class="stat-label">Régions couvertes</div>
    </div>
    <div class="stat-item">
      <div class="stat-num">10+</div>
      <div class="stat-label">Variables analysées</div>
    </div>
    <div class="stat-item">
      <div class="stat-num">100%</div>
      <div class="stat-label">Anonymat garanti</div>
    </div>
  </div>
</section>

<!-- FONCTIONNALITÉS -->
<section class="features">
  <h2 class="features-title">Pourquoi AgroSense ?</h2>
  <div class="features-grid">
    <div class="feature-card">
      <div class="feature-icon fi-g">📋</div>
      <h3>Collecte structurée</h3>
      <p>Un formulaire en 5 sections couvrant le profil, l'exploitation, la production, les défis et les prix. Validation en temps réel.</p>
    </div>
    <div class="feature-card">
      <div class="feature-icon fi-o">📊</div>
      <h3>Analyse descriptive</h3>
      <p>7 graphiques interactifs — barres, courbes, doughnut, scatter — calculés automatiquement à partir des données soumises.</p>
    </div>
    <div class="feature-card">
      <div class="feature-icon fi-y">💡</div>
      <h3>Insights intelligents</h3>
      <p>L'application génère des recommandations basées sur les seuils FAO et les tendances détectées dans les données.</p>
    </div>
    <div class="feature-card">
      <div class="feature-icon fi-b">🔒</div>
      <h3>Données anonymes</h3>
      <p>Aucune donnée personnelle identifiable n'est collectée. La confidentialité des agriculteurs est garantie.</p>
    </div>
  </div>
</section>

<!-- COMMENT ÇA MARCHE -->
<section class="how">
  <h2 class="how-title">Comment ça marche ?</h2>
  <div class="steps">
    <div class="step">
      <div class="step-num">01</div>
      <h4>Remplissez le sondage</h4>
      <p>Répondez aux questions sur votre exploitation agricole en 3 à 5 minutes.</p>
    </div>
    <div class="step">
      <div class="step-num">02</div>
      <h4>Données enregistrées</h4>
      <p>Vos réponses sont stockées en base de données SQLite de façon sécurisée et anonyme.</p>
    </div>
    <div class="step">
      <div class="step-num">03</div>
      <h4>Analysez les résultats</h4>
      <p>Le dashboard se met à jour automatiquement avec chaque nouvelle réponse collectée.</p>
    </div>
    <div class="step">
      <div class="step-num">04</div>
      <h4>Agissez sur les insights</h4>
      <p>Les recommandations générées aident chercheurs et institutions à orienter leurs actions.</p>
    </div>
  </div>
</section>

<!-- CTA FINAL -->
<section class="cta">
  <h2>Prêt à contribuer ?</h2>
  <p>Chaque réponse compte pour améliorer la compréhension de l'agriculture camerounaise.</p>
  <a href="{{ url_for('sondage') }}" class="btn-primary">Accéder à la collecte →</a>
</section>

{% endblock %}


{% extends "base.html" %}
{% block title %}Sondage — AgroSense Cameroun{% endblock %}

{% block extra_css %}
<style>
.form-wrap { max-width: 720px; margin: 0 auto; padding: 48px 24px 80px; }
.form-hd { margin-bottom: 40px; }
.form-hd h2 { font-family:'Playfair Display',serif; font-size:42px; font-weight:700; margin-bottom:10px; }
.form-hd p  { color:var(--muted); font-size:14px; line-height:1.7; }

.prog-bar { height:3px; background:var(--surf3); border-radius:10px; margin-bottom:36px; overflow:hidden; }
.prog-fill { height:100%; background:linear-gradient(90deg,var(--green),var(--orange)); border-radius:10px; transition:width .4s ease; width:0%; }
.prog-label { text-align:right; font-size:12px; color:var(--muted); margin-bottom:6px; font-weight:500; }

.form-section {
  background:var(--surf); border:1px solid var(--border);
  border-radius:var(--r); padding:28px; margin-bottom:16px;
  transition:border-color .25s;
}
.form-section:hover { border-color:rgba(74,222,128,.18); }
.sec-hd { display:flex; align-items:center; gap:12px; margin-bottom:24px; }
.sec-icon { width:38px; height:38px; border-radius:10px; display:flex; align-items:center; justify-content:center; font-size:18px; flex-shrink:0; }
.ic-a { background:rgba(74,222,128,.1); }
.ic-b { background:rgba(251,146,60,.1); }
.ic-c { background:rgba(251,191,36,.1); }
.ic-d { background:rgba(147,197,253,.1); }
.ic-e { background:rgba(216,180,254,.1); }
.sec-hd h3 { font-size:16px; font-weight:700; }

.form-group { margin-bottom:20px; }
.form-row { display:grid; grid-template-columns:1fr 1fr; gap:16px; }
.form-label { display:block; font-size:13px; font-weight:600; margin-bottom:8px; letter-spacing:.1px; }
.form-label .req { color:#f87171; margin-left:2px; }

.form-input, .form-select, .form-textarea {
  width:100%; padding:12px 14px;
  background:var(--surf2); border:1px solid var(--border);
  border-radius:10px; color:var(--text);
  font-family:'DM Sans',sans-serif; font-size:13px;
  outline:none; transition:border-color .2s;
}
.form-input:focus, .form-select:focus, .form-textarea:focus { border-color:var(--green); }
.form-select {
  appearance:none; padding-right:36px;
  background-image:url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='8'%3E%3Cpath d='M1 1l5 5 5-5' stroke='%236b7c70' stroke-width='1.5' fill='none' stroke-linecap='round'/%3E%3C/svg%3E");
  background-repeat:no-repeat; background-position:right 13px center;
}
.form-select option { background:var(--surf2); }
.form-textarea { resize:vertical; min-height:90px; }

.range-input {
  width:100%; -webkit-appearance:none;
  height:3px; border-radius:10px;
  background:var(--surf3); outline:none; margin:10px 0; cursor:pointer;
}
.range-input::-webkit-slider-thumb {
  -webkit-appearance:none; width:20px; height:20px; border-radius:50%;
  background:var(--green); cursor:pointer;
  box-shadow:0 0 0 4px rgba(74,222,128,.18);
}
.range-val {
  text-align:center; font-family:'Playfair Display',serif;
  font-size:28px; font-weight:700; color:var(--green); margin-top:4px;
}
.range-labels { display:flex; justify-content:space-between; font-size:11px; color:var(--muted); margin-top:2px; }

/* ── CHECKBOXES CORRIGÉES ── */
.chk-grid { display:grid; grid-template-columns:repeat(auto-fill,minmax(200px,1fr)); gap:8px; }
.chk-item {
  display:flex; align-items:center; gap:10px;
  padding:10px 13px; background:var(--surf2);
  border:1px solid var(--border); border-radius:9px;
  cursor:pointer; font-size:13px; font-weight:500;
  transition:all .15s; user-select:none;
}
.chk-item:hover { border-color:var(--green); }
/* On cache l'input NATIF mais on l'utilise pour le submit */
.chk-item input[type="checkbox"] {
  position: absolute; opacity: 0; width: 0; height: 0; pointer-events: none;
}
.chk-item.checked { border-color:var(--green); background:rgba(74,222,128,.06); color:var(--green); }
.chk-box {
  width:17px; height:17px; border-radius:5px;
  border:1px solid var(--muted); flex-shrink:0;
  display:flex; align-items:center; justify-content:center;
  font-size:11px; font-weight:900; transition:all .15s;
  background: var(--surf3); color: transparent;
}
.chk-item.checked .chk-box {
  background:var(--green); border-color:var(--green);
  color:#0d1a0f;
}

.submit-row {
  display:flex; align-items:center; justify-content:space-between;
  margin-top:28px; flex-wrap:wrap; gap:14px;
}
.privacy-note { font-size:12px; color:var(--muted); display:flex; align-items:center; gap:6px; }
</style>
{% endblock %}

{% block content %}
<div class="form-wrap">
  <div class="form-hd fade-up">
    <h2>🌱 Sondage Agricole</h2>
    <p>Formulaire anonyme de collecte de données agricoles camerounaises. Durée estimée : 3 à 5 minutes. Les champs marqués <span style="color:#f87171">*</span> sont obligatoires.</p>
  </div>

  <div class="prog-label"><span id="prog-pct">0%</span> complété</div>
  <div class="prog-bar"><div class="prog-fill" id="prog-fill"></div></div>

  <form method="POST" action="{{ url_for('sondage_submit') }}" id="survey-form" novalidate>

    <!-- SECTION 1 : PROFIL -->
    <div class="form-section fade-up">
      <div class="sec-hd">
        <div class="sec-icon ic-a">👤</div>
        <h3>Profil de l'agriculteur</h3>
      </div>

      <div class="form-row">
        <div class="form-group">
          <label class="form-label">Région <span class="req">*</span></label>
          <select name="region" class="form-select" required>
            <option value="">-- Sélectionner --</option>
            {% for r in ["Adamaoua","Centre","Est","Extrême-Nord","Littoral","Nord","Nord-Ouest","Ouest","Sud","Sud-Ouest"] %}
            <option value="{{ r }}">{{ r }}</option>
            {% endfor %}
          </select>
        </div>
        <div class="form-group">
          <label class="form-label">Département</label>
          <input type="text" name="departement" class="form-input" placeholder="Ex: Mfoundi"/>
        </div>
      </div>

      <div class="form-row">
        <div class="form-group">
          <label class="form-label">Statut <span class="req">*</span></label>
          <select name="statut" class="form-select" required>
            <option value="">-- Sélectionner --</option>
            <option value="petit-exploitant">Petit exploitant (< 2 ha)</option>
            <option value="moyen-exploitant">Moyen exploitant (2–10 ha)</option>
            <option value="grand-exploitant">Grand exploitant (> 10 ha)</option>
            <option value="cooperateur">Membre d'une coopérative</option>
            <option value="agronome">Agronome / Technicien</option>
          </select>
        </div>
        <div class="form-group">
          <label class="form-label">Genre <span class="req">*</span></label>
          <select name="genre" class="form-select" required>
            <option value="">-- Sélectionner --</option>
            <option value="homme">Homme</option>
            <option value="femme">Femme</option>
            <option value="nsp">Préfère ne pas répondre</option>
          </select>
        </div>
      </div>

      <div class="form-group">
        <label class="form-label">Tranche d'âge <span class="req">*</span></label>
        <select name="age" class="form-select" required>
          <option value="">-- Sélectionner --</option>
          <option value="moins-25">Moins de 25 ans</option>
          <option value="25-34">25 – 34 ans</option>
          <option value="35-44">35 – 44 ans</option>
          <option value="45-54">45 – 54 ans</option>
          <option value="55+">55 ans et plus</option>
        </select>
      </div>
    </div>

    <!-- SECTION 2 : EXPLOITATION -->
    <div class="form-section">
      <div class="sec-hd">
        <div class="sec-icon ic-b">🌿</div>
        <h3>Exploitation agricole</h3>
      </div>

      <div class="form-group">
        <label class="form-label">Cultures pratiquées <span class="req">*</span>
          <small style="color:var(--muted);font-weight:400"> — cliquez pour sélectionner</small>
        </label>
        <div class="chk-grid" id="cultures-group">
          {% for c, v in [
            ("Maïs","mais"), ("Manioc","manioc"), ("Cacao","cacao"),
            ("Café","cafe"), ("Plantain","plantain"), ("Igname","igname"),
            ("Sorgho","sorgho"), ("Mil","mil"), ("Arachide","arachide"),
            ("Palmier à huile","palmier"), ("Riz","riz"), ("Tomate","tomate"),
            ("Piment","piment"), ("Haricot","haricot")
          ] %}
          <div class="chk-item" data-value="{{ v }}">
            <input type="checkbox" name="cultures" value="{{ v }}" id="cult-{{ v }}"/>
            <div class="chk-box">✓</div>
            <span>{{ c }}</span>
          </div>
          {% endfor %}
        </div>
      </div>

      <div class="form-row">
        <div class="form-group">
          <label class="form-label">Surface exploitée (hectares) <span class="req">*</span></label>
          <input type="range" class="range-input" name="surface" id="surface-range"
                 min="0.1" max="50" step="0.1" value="2"
                 oninput="updateRange('surface-range','surface-val',' ha')"/>
          <div class="range-val" id="surface-val">2 ha</div>
          <div class="range-labels"><span>0.1 ha</span><span>50 ha</span></div>
        </div>
        <div class="form-group">
          <label class="form-label">Mode de culture <span class="req">*</span></label>
          <select name="mode_culture" class="form-select" required>
            <option value="">-- Sélectionner --</option>
            <option value="traditionnel">Traditionnel (manuel)</option>
            <option value="semi-mecanise">Semi-mécanisé</option>
            <option value="mecanise">Mécanisé</option>
            <option value="agroforesterie">Agroforesterie</option>
            <option value="biologique">Agriculture biologique</option>
          </select>
        </div>
      </div>
    </div>

    <!-- SECTION 3 : PRODUCTION -->
    <div class="form-section">
      <div class="sec-hd">
        <div class="sec-icon ic-c">📦</div>
        <h3>Production</h3>
      </div>

      <div class="form-group">
        <label class="form-label">Rendement estimé (kg/ha) <span class="req">*</span></label>
        <input type="range" class="range-input" name="rendement" id="rendement-range"
               min="50" max="5000" step="50" value="800"
               oninput="updateRange('rendement-range','rendement-val',' kg/ha')"/>
        <div class="range-val" id="rendement-val">800 kg/ha</div>
        <div class="range-labels"><span>50 kg/ha</span><span>5 000 kg/ha</span></div>
      </div>

      <div class="form-row">
        <div class="form-group">
          <label class="form-label">Saison de récolte <span class="req">*</span></label>
          <select name="saison" class="form-select" required>
            <option value="">-- Sélectionner --</option>
            <option value="grande-saison">Grande saison sèche</option>
            <option value="grande-pluie">Grande saison des pluies</option>
            <option value="petite-saison">Petite saison sèche</option>
            <option value="petite-pluie">Petite saison des pluies</option>
            <option value="annuel">Production annuelle</option>
          </select>
        </div>
        <div class="form-group">
          <label class="form-label">Année de récolte <span class="req">*</span></label>
          <select name="annee" class="form-select" required>
            <option value="">-- Sélectionner --</option>
            {% for y in range(2026, 2019, -1) %}
            <option value="{{ y }}">{{ y }}</option>
            {% endfor %}
          </select>
        </div>
      </div>
    </div>

    <!-- SECTION 4 : DÉFIS -->
    <div class="form-section">
      <div class="sec-hd">
        <div class="sec-icon ic-d">⚠️</div>
        <h3>Défis rencontrés</h3>
      </div>
      <div class="form-group">
        <label class="form-label">Principaux défis
          <small style="color:var(--muted);font-weight:400"> — optionnel, plusieurs choix possibles</small>
        </label>
        <div class="chk-grid" id="defis-group">
          {% for label, val in [
            ("Maladies des plantes","maladies"),
            ("Ravageurs / insectes","ravageurs"),
            ("Sécheresse","secheresse"),
            ("Inondations","inondations"),
            ("Manque d'engrais","engrais"),
            ("Accès difficile au marché","marche"),
            ("Financement / crédit","financement"),
            ("Main-d'œuvre insuffisante","main_oeuvre"),
            ("Prix de vente bas","prix_bas"),
            ("Vols de récoltes","vols")
          ] %}
          <div class="chk-item" data-value="{{ val }}">
            <input type="checkbox" name="defis" value="{{ val }}" id="defi-{{ val }}"/>
            <div class="chk-box">✓</div>
            <span>{{ label }}</span>
          </div>
          {% endfor %}
        </div>
      </div>
    </div>

    <!-- SECTION 5 : MARCHÉ -->
    <div class="form-section">
      <div class="sec-hd">
        <div class="sec-icon ic-e">💰</div>
        <h3>Marché &amp; Prix</h3>
      </div>

      <div class="form-group">
        <label class="form-label">Prix de vente moyen (F CFA / kg)</label>
        <input type="range" class="range-input" name="prix_vente" id="prix-range"
               min="0" max="2000" step="10" value="300"
               oninput="updateRange('prix-range','prix-val',' F CFA/kg')"/>
        <div class="range-val" id="prix-val">300 F CFA/kg</div>
        <div class="range-labels"><span>0 F CFA</span><span>2 000 F CFA</span></div>
      </div>

      <div class="form-row">
        <div class="form-group">
          <label class="form-label">Canal de distribution</label>
          <select name="canal_distrib" class="form-select">
            <option value="">-- Sélectionner --</option>
            <option value="marche-local">Marché local</option>
            <option value="marche-regional">Marché régional</option>
            <option value="grossiste">Grossiste / Collecteur</option>
            <option value="cooperative">Coopérative</option>
            <option value="export">Export</option>
            <option value="autoconsommation">Autoconsommation</option>
          </select>
        </div>
        <div class="form-group">
          <label class="form-label">Distance au marché le plus proche</label>
          <select name="distance_marche" class="form-select">
            <option value="">-- Sélectionner --</option>
            <option value="moins-5km">Moins de 5 km</option>
            <option value="5-20km">5 – 20 km</option>
            <option value="20-50km">20 – 50 km</option>
            <option value="plus-50km">Plus de 50 km</option>
          </select>
        </div>
      </div>
    </div>

    <!-- COMMENTAIRE -->
    <div class="form-section">
      <div class="sec-hd">
        <div class="sec-icon ic-a">💬</div>
        <h3>Commentaire libre (optionnel)</h3>
      </div>
      <div class="form-group">
        <label class="form-label">Partagez tout ce que vous souhaitez ajouter</label>
        <textarea name="commentaire" class="form-textarea"
                  placeholder="Vos observations, suggestions, difficultés spécifiques..."></textarea>
      </div>
    </div>

    <div class="submit-row">
      <div class="privacy-note">🔒 Données anonymes — aucune info personnelle identifiable</div>
      <button type="submit" class="btn-primary">Soumettre mes données →</button>
    </div>
  </form>
</div>
{% endblock %}

{% block extra_js %}
<script>
// ── RANGE ─────────────────────────────────────────────────────────────────────
function updateRange(inputId, valId, unit) {
  const v = document.getElementById(inputId).value;
  document.getElementById(valId).textContent = parseFloat(v).toLocaleString('fr-FR') + unit;
  updateProgress();
}

// ── CHECKBOXES (VERSION CORRIGÉE) ────────────────────────────────────────────
// On attache le clic directement sur le div.chk-item, PAS sur le <label>
// pour éviter le double déclenchement natif du navigateur.
document.querySelectorAll('.chk-item').forEach(function(item) {
  item.addEventListener('click', function(e) {
    e.preventDefault();   // empêche tout comportement par défaut
    e.stopPropagation();

    const input = item.querySelector('input[type="checkbox"]');
    if (!input) return;

    // Inverse l'état
    const isNowChecked = !input.checked;
    input.checked = isNowChecked;

    // Synchronise l'apparence
    if (isNowChecked) {
      item.classList.add('checked');
    } else {
      item.classList.remove('checked');
    }

    updateProgress();
  });
});

// ── PROGRESS ──────────────────────────────────────────────────────────────────
function updateProgress() {
  const requiredSelects = ['region', 'statut', 'genre', 'age', 'mode_culture', 'saison', 'annee'];
  let filled = 0;

  requiredSelects.forEach(function(name) {
    const el = document.querySelector('select[name="' + name + '"]');
    if (el && el.value) filled++;
  });

  // Cultures : au moins 1 cochée
  const culturesChecked = document.querySelectorAll('input[name="cultures"]:checked');
  if (culturesChecked.length > 0) filled++;

  const total = requiredSelects.length + 1;
  const pct   = Math.round(filled / total * 100);
  document.getElementById('prog-fill').style.width = pct + '%';
  document.getElementById('prog-pct').textContent  = pct + '%';
}

// Écouter les selects pour la progression
document.querySelectorAll('select').forEach(function(sel) {
  sel.addEventListener('change', updateProgress);
});

// Init
updateProgress();
</script>
{% endblock %}


{% extends "base.html" %}
{% block title %}Merci ! — AgroSense Cameroun{% endblock %}
{% block extra_css %}
<style>
.success-page {
  min-height: calc(100vh - 62px);
  display: flex; align-items: center; justify-content: center;
  padding: 60px 24px;
}
.success-card {
  text-align: center; max-width: 520px; width: 100%;
  background: var(--surf); border: 1px solid var(--border);
  border-radius: 20px; padding: 56px 40px;
}
.success-circle {
  width: 88px; height: 88px; border-radius: 50%;
  background: rgba(74,222,128,.08); border: 2px solid var(--green);
  display: flex; align-items: center; justify-content: center;
  font-size: 38px; margin: 0 auto 28px;
  animation: pulse 2s infinite;
}
@keyframes pulse { 0%,100%{box-shadow:0 0 0 0 rgba(74,222,128,.3)} 50%{box-shadow:0 0 0 16px rgba(74,222,128,0)} }
.success-card h1 {
  font-family:'Playfair Display',serif;
  font-size:32px; font-weight:700; margin-bottom:14px;
}
.success-card p { color:var(--muted); font-size:15px; line-height:1.7; margin-bottom:10px; }
.count-box {
  background: var(--surf2); border-radius: 12px;
  padding: 16px 24px; margin: 24px 0;
  display: flex; align-items: center; justify-content: center; gap: 14px;
}
.count-num {
  font-family:'Playfair Display',serif;
  font-size: 44px; font-weight:900; color:var(--green); line-height:1;
}
.count-label { font-size:13px; color:var(--muted); text-align:left; }
.success-btns { display:flex; gap:12px; justify-content:center; margin-top:28px; flex-wrap:wrap; }
</style>
{% endblock %}

{% block content %}
<div class="success-page">
  <div class="success-card fade-up">
    <div class="success-circle">✓</div>
    <h1>Merci pour votre contribution !</h1>
    <p>Vos données agricoles ont été enregistrées avec succès dans notre base de données. Elles contribuent à l'amélioration de la connaissance du secteur agricole camerounais.</p>

    <div class="count-box">
      <div class="count-num">{{ total }}</div>
      <div class="count-label">réponse{{ 's' if total > 1 else '' }} collectée{{ 's' if total > 1 else '' }}<br/>au total</div>
    </div>

    <p style="font-size:13px">Le tableau de bord a été mis à jour automatiquement avec vos nouvelles données.</p>

    <div class="success-btns">
      <a href="{{ url_for('dashboard') }}" class="btn-primary">📊 Voir les résultats →</a>
      <a href="{{ url_for('sondage') }}"   class="btn-secondary">Collecter à nouveau</a>
    </div>
  </div>
</div>
{% endblock %}
