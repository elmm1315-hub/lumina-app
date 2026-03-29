# lumina-app<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Lumina | Mind & Heart Map</title>
    <link rel="manifest" href="manifest.json">
    <script src="https://d3js.org/d3.v7.min.js"></script>
    <style>
        :root { 
            --bg: #0b0e14; --card: #161b22; --primary: #6c5ce7; 
            --text: #c9d1d9; --empathy: #ff7675; --knowledge: #74b9ff; --accent: #f1c40f;
        }
        body { 
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Helvetica, Arial, sans-serif; 
            background: var(--bg); color: var(--text); margin: 0; padding: 20px;
            display: flex; flex-direction: column; align-items: center;
        }
        .container { width: 100%; max-width: 500px; }
        .card { 
            background: var(--card); border: 1px solid #30363d; border-radius: 12px; 
            padding: 20px; margin-bottom: 20px; box-shadow: 0 4px 12px rgba(0,0,0,0.5);
        }
        h1, h2, h3 { margin-top: 0; color: #fff; }
        
        /* Search & Input */
        input, textarea { 
            width: 100%; padding: 12px; background: #0d1117; border: 1px solid #30363d; 
            border-radius: 8px; color: #fff; margin-bottom: 12px; box-sizing: border-box;
        }
        button { 
            width: 100%; padding: 12px; border: none; border-radius: 8px; 
            font-weight: 600; cursor: pointer; transition: 0.2s; margin-bottom: 8px;
        }
        .btn-main { background: var(--primary); color: #fff; }
        .btn-outline { background: transparent; border: 1px solid var(--primary); color: var(--primary); }
        
        /* Book Preview */
        #book-preview { display: none; border-top: 1px solid #30363d; pt: 15px; margin-top: 10px; }
        .book-meta { display: flex; gap: 15px; align-items: start; }
        .book-meta img { width: 70px; border-radius: 4px; box-shadow: 0 2px 8px rgba(0,0,0,0.4); }
        .tags { display: flex; gap: 5px; flex-wrap: wrap; margin: 8px 0; }
        .tag { font-size: 10px; padding: 2px 6px; background: #21262d; border-radius: 4px; border: 1px solid #30363d; }

        /* Graph Area */
        #graph-wrapper { 
            width: 100%; height: 350px; background: #0d1117; border-radius: 12px; 
            border: 1px solid #30363d; position: relative; overflow: hidden;
        }
        
        /* Achievements & Stats */
        .stats-bar { display: flex; justify-content: space-between; font-size: 12px; margin-bottom: 10px; }
        .badges { display: flex; justify-content: center; gap: 12px; margin: 15px 0; }
        .badge { font-size: 20px; opacity: 0.2; transition: 0.5s; filter: grayscale(1); }
        .badge.unlocked { opacity: 1; filter: grayscale(0); transform: scale(1.2); }

        #toast { 
            position: fixed; bottom: 20px; background: var(--primary); color: #fff; 
            padding: 12px 24px; border-radius: 50px; display: none; animation: slideUp 0.3s;
        }
        @keyframes slideUp { from { transform: translateY(100px); } to { transform: translateY(0); } }
    </style>
</head>
<body>

<div class="container">
    <div class="stats-bar">
        <span id="streak-display">🔥 0 Day Streak</span>
        <span id="book-count">📚 0 Books</span>
    </div>

    <div class="card">
        <h3>Log a Journey</h3>
        <input type="text" id="searchInput" placeholder="Search book title or author..." onchange="searchBooks()">
        
        <div id="book-preview">
            <div class="book-meta">
                <img id="prev-cover" src="" alt="cover">
                <div>
                    <h4 id="prev-title" style="margin:0;"></h4>
                    <div class="tags" id="prev-tags"></div>
                </div>
            </div>
            <p id="prev-desc" style="font-size: 11px; color: #8b949e; margin: 10px 0;"></p>
            <textarea id="userThoughts" placeholder="What did this book spark in you?"></textarea>
            <button class="btn-main" onclick="logFinalBook()">Complete Evolution</button>
        </div>
    </div>

    <div id="graph-wrapper">
        <svg id="mindsat-graph" style="width:100%; height:100%"></svg>
    </div>

    <div class="badges">
        <span id="badge-empath" class="badge" title="The Empath">❤️</span>
        <span id="badge-scholar" class="badge" title="Dedicated Scholar">📜</span>
        <span id="badge-polymath" class="badge" title="Polymath">🧠</span>
    </div>

    <button class="btn-outline" onclick="exportMap()">📸 Export Knowledge Map</button>
    <button style="background:transparent; color:#484f58; font-size:10px;" onclick="resetLumina()">Reset All Data</button>
</div>

<div id="toast">Map Updated!</div>
<canvas id="exportCanvas" style="display:none"></canvas>

<script>
    // --- APP STATE ---
    let tempBook = null;
    let userProfile = JSON.parse(localStorage.getItem('lumina_v1_data')) || {
        history: [],
        badges: [],
        streak: 0,
        lastDate: null,
        nodes: [
            { id: "Self", r: 18, color: "#fff", type: "core" },
            { id: "Empathy", r: 14, color: "#ff7675", type: "root" },
            { id: "Knowledge", r: 14, color: "#74b9ff", type: "root" }
        ],
        links: [
            { source: "Self", target: "Empathy" },
            { source: "Self", target: "Knowledge" }
        ]
    };

    // --- SEARCH & API ---
    async function searchBooks() {
        const query = document.getElementById('searchInput').value;
        const res = await fetch(`https://www.googleapis.com/books/v1/volumes?q=${query}`);
        const data = await res.json();
        if(data.items) {
            const b = data.items[0].volumeInfo;
            tempBook = b;
            document.getElementById('prev-title').innerText = b.title;
            document.getElementById('prev-cover').src = b.imageLinks?.thumbnail || '';
            document.getElementById('prev-desc').innerText = b.description?.substring(0, 120) + "...";
            document.getElementById('prev-tags').innerHTML = `
                <span class="tag">⭐ ${b.averageRating || 'N/A'}</span>
                <span class="tag">📄 ${b.pageCount || '???'}pg</span>
                <span class="tag">${b.categories?.[0] || 'General'}</span>
            `;
            document.getElementById('book-preview').style.display = 'block';
        }
    }

    // --- CORE LOGIC ---
    function logFinalBook() {
        const thoughts = document.getElementById('userThoughts').value;
        const isFiction = tempBook.categories?.some(c => c.toLowerCase().includes('fiction'));
        
        // Empathy Analysis
        const keywords = ['feel', 'felt', 'human', 'understand', 'perspective', 'pain', 'joy'];
        let score = 20; 
        keywords.forEach(w => { if(thoughts.toLowerCase().includes(w)) score += 15 });

        // Update Graph Data
        const newNode = { 
            id: tempBook.title, 
            r: 8, 
            color: isFiction ? "#fab1a0" : "#81ecec",
            type: "book"
        };
        userProfile.nodes.push(newNode);
        userProfile.links.push({ 
            source: isFiction ? "Empathy" : "Knowledge", 
            target: tempBook.title 
        });

        // Update History & Streak
        userProfile.history.push({ title: tempBook.title, date: new Date().toISOString() });
        handleStreak();
        checkBadges(isFiction);
        
        saveAndRender();
        triggerToast();
        document.getElementById('book-preview').style.display = 'none';
        document.getElementById('searchInput').value = '';
    }

    function handleStreak() {
        const today = new Date().toDateString();
        if(userProfile.lastDate !== today) {
            userProfile.streak = (userProfile.lastDate === new Date(Date.now() - 86400000).toDateString()) ? userProfile.streak + 1 : 1;
            userProfile.lastDate = today;
        }
    }

    function checkBadges(isFiction) {
        if(isFiction && !userProfile.badges.includes('empath')) userProfile.badges.push('empath');
        if(userProfile.history.length >= 5 && !userProfile.badges.includes('polymath')) userProfile.badges.push('polymath');
        if(userProfile.streak >= 3 && !userProfile.badges.includes('scholar')) userProfile.badges.push('scholar');
    }

    // --- VISUALIZATION ---
    let svg = d3.select("#mindsat-graph");
    let simulation = d3.forceSimulation()
        .force("link", d3.forceLink().id(d => d.id).distance(50))
        .force("charge", d3.forceManyBody().strength(-100))
        .force("center", d3.forceCenter(window.innerWidth > 500 ? 250 : 200, 175));

    function renderGraph() {
        svg.selectAll("*").remove();
        
        const link = svg.append("g").attr("stroke", "#30363d").selectAll("line")
            .data(userProfile.links).enter().append("line");

        const node = svg.append("g").selectAll("circle")
            .data(userProfile.nodes).enter().append("circle")
            .attr("r", d => d.r)
            .attr("fill", d => d.color)
            .attr("stroke", d => d.type === 'core' ? varToHex('--primary') : "none")
            .attr("stroke-width", 2);

        simulation.nodes(userProfile.nodes);
        simulation.force("link").links(userProfile.links);
        simulation.on("tick", () => {
            link.attr("x1", d => d.source.x).attr("y1", d => d.source.y)
                .attr("x2", d => d.target.x).attr("y2", d => d.target.y);
            node.attr("cx", d => d.x).attr("cy", d => d.y);
        });
        simulation.alpha(1).restart();
    }

    // --- UTILS ---
    function saveAndRender() {
        localStorage.setItem('lumina_v1_data', JSON.stringify(userProfile));
        renderGraph();
        updateUI();
    }

    function updateUI() {
        document.getElementById('streak-display').innerText = `🔥 ${userProfile.streak} Day Streak`;
        document.getElementById('book-count').innerText = `📚 ${userProfile.history.length} Books`;
        userProfile.badges.forEach(b => document.getElementById(`badge-${b}`).classList.add('unlocked'));
    }

    function triggerToast() {
        const t = document.getElementById('toast');
        t.style.display = 'block';
        setTimeout(() => t.style.display = 'none', 2000);
    }

    function exportMap() {
        const svgEl = document.getElementById('mindsat-graph');
        const canvas = document.getElementById('exportCanvas');
        const svgData = new XMLSerializer().serializeToString(svgEl);
        const img = new Image();
        img.onload = () => {
            canvas.width = 800; canvas.height = 800;
            const ctx = canvas.getContext('2d');
            ctx.fillStyle = "#0b0e14"; ctx.fillRect(0,0,800,800);
            ctx.drawImage(img, 0, 0, 800, 800);
            const a = document.createElement('a');
            a.download = 'lumina-map.png'; a.href = canvas.toDataURL(); a.click();
        };
        img.src = 'data:image/svg+xml;base64,' + btoa(svgData);
    }

    function resetLumina() { if(confirm("Erase your mind map?")) { localStorage.clear(); location.reload(); } }
    function varToHex(v) { return getComputedStyle(document.body).getPropertyValue(v).trim(); }

    window.onload = () => { renderGraph(); updateUI(); };
</script>
</body>
</html>
{
  "name": "Lumina: Mind & Heart",
  "short_name": "Lumina",
  "start_url": "index.html",
  "display": "standalone",
  "background_color": "#0b0e14",
  "theme_color": "#6c5ce7",
  "icons": [
    {
      "src": "https://via.placeholder.com/192/6c5ce7/ffffff?text=L",
      "sizes": "192x192",
      "type": "image/png"
    }
  ]
}
