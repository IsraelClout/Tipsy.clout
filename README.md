<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Tipsy.clout</title>
<style>
  :root { --bg:#0c0c0c; --card:#1a1a1a; --accent:#ff5722; }
  *{box-sizing:border-box;margin:0;padding:0;font-family:Segoe UI,sans-serif}
  body{background:var(--bg);color:#fff}

  header{
    position:sticky;top:0;z-index:10;background:rgba(12,12,12,0.9);
    backdrop-filter:blur(8px);padding:16px;border-bottom:1px solid #222
  }
 .wrap{max-width:1200px;margin:0 auto;padding:0 16px}
  h1{font-size:22px;margin-bottom:12px;color:var(--accent)}
 .search-box{display:flex;gap:8px}
 .search-box input{
    flex:1;padding:12px 16px;border-radius:12px;border:1px solid #333;
    background:var(--card);color:#fff;font-size:16px;outline:none
  }
 .search-box button{
    padding:12px 20px;border-radius:12px;border:0;background:var(--accent);
    color:#fff;font-weight:600;cursor:pointer
  }

 .tabs{display:flex;gap:12px;margin:16px 0}
 .tab{
    padding:8px 16px;border-radius:20px;background:var(--card);border:1px solid #333;
    cursor:pointer;font-size:14px
  }
 .tab.active{background:var(--accent);border-color:var(--accent)}

 .grid{
    display:grid;grid-template-columns:repeat(auto-fill,minmax(280px,1fr));
    gap:16px;padding:16px 0
  }
 .card{
    background:var(--card);border-radius:16px;overflow:hidden;border:1px solid #222;
    transition:.2s
  }
 .card:hover{transform:translateY(-4px);border-color:#333}
 .card img,.card video{width:100%;height:200px;object-fit:cover;display:block}
 .card-body{padding:12px}
 .card-body p{font-size:14px;color:#aaa;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}
 .loader{text-align:center;padding:40px;color:#888}
 .error{text-align:center;padding:40px;color:#ff6b6b}
</style>
</head>
<body>

<header>
  <div class="wrap">
    <h1>Tipsy 🔥</h1>
    <div class="search-box">
      <input type="text" id="searchInput" placeholder="Search images or videos... e.g. nature, cars, tech">
      <button id="searchBtn">Search</button>
    </div>
    <div class="tabs">
      <div class="tab active" data-type="images">Images</div>
      <div class="tab" data-type="videos">Videos</div>
    </div>
  </div>
</header>

<main class="wrap">
  <div id="grid" class="grid"></div>
  <div id="loader" class="loader" style="display:none">Loading...</div>
  <div id="error" class="error" style="display:none"></div>
</main>

<script>
  // ===== CONFIG =====
  const PEXELS_KEY = "zZHKyQVSb9mGlvZ4R9AoYrp98qQdztMtr2tXka3JflOjcWlfDWg9N83C"; // <- Your API key
  const PER_PAGE = 20;

  // ===== STATE =====
  let currentType = 'images';
  let currentQuery = 'trending';
  let page = 1;

  const grid = document.getElementById('grid');
  const loader = document.getElementById('loader');
  const errorBox = document.getElementById('error');
  const searchInput = document.getElementById('searchInput');
  const searchBtn = document.getElementById('searchBtn');

  // ===== TABS =====
  document.querySelectorAll('.tab').forEach(tab => {
    tab.onclick = () => {
      document.querySelectorAll('.tab').forEach(t => t.classList.remove('active'));
      tab.classList.add('active');
      currentType = tab.dataset.type;
      page = 1;
      fetchMedia();
    }
  });

  // ===== SEARCH =====
  searchBtn.onclick = () => {
    currentQuery = searchInput.value.trim() || 'trending';
    page = 1;
    fetchMedia();
  }
  searchInput.addEventListener('keypress', e => {
    if(e.key === 'Enter') searchBtn.click();
  });

  // ===== API FETCH =====
  async function fetchMedia() {
    grid.innerHTML = '';
    loader.style.display = 'block';
    errorBox.style.display = 'none';

    try {
      if(currentType === 'images') {
        // Picsum API - no key needed. We filter client-side for demo search
        const res = await fetch(`https://picsum.photos/v2/list?page=${page}&limit=${PER_PAGE}`);
        const data = await res.json();
        const filtered = currentQuery === 'trending'? data :
          data.filter(i => i.author.toLowerCase().includes(currentQuery.toLowerCase()));
        renderImages(filtered);
      }
      else {
        // Pexels Videos API - uses your key
        const res = await fetch(`https://api.pexels.com/videos/search?query=${currentQuery}&per_page=${PER_PAGE}&page=${page}`, {
          headers: { Authorization: PEXELS_KEY }
        });
        if(!res.ok) throw new Error('API error: ' + res.status);
        const data = await res.json();
        renderVideos(data.videos);
      }
    } catch(err) {
      errorBox.textContent = 'Error: ' + err.message;
      errorBox.style.display = 'block';
    } finally {
      loader.style.display = 'none';
    }
  }

  // ===== RENDER =====
  function renderImages(items) {
    if(items.length === 0) {
      grid.innerHTML = '<p class="loader">No images found</p>';
      return;
    }
    grid.innerHTML = items.map(i => `
      <div class="card">
        <img src="${i.download_url}" loading="lazy" alt="${i.author}">
        <div class="card-body">
          <p>By: ${i.author}</p>
        </div>
      </div>
    `).join('');
  }

  function renderVideos(videos) {
    if(videos.length === 0) {
      grid.innerHTML = '<p class="loader">No videos found</p>';
      return;
    }
    grid.innerHTML = videos.map(v => {
      const file = v.video_files.find(f => f.quality === 'hd') || v.video_files[0];
      return `
        <div class="card">
          <video src="${file.link}" controls muted loop playsinline poster="${v.image}"></video>
          <div class="card-body">
            <p>By: ${v.user.name}</p>
          </div>
        </div>
      `;
    }).join('');
  }

  // ===== INIT =====
  fetchMedia(); // load trending on start

  // ===== Infinite Scroll =====
  let loadingMore = false;
  window.addEventListener('scroll', () => {
    if(loadingMore) return;
    if(window.innerHeight + window.scrollY >= document.body.offsetHeight - 800) {
      loadingMore = true;
      page++;
      fetchMediaAppend();
    }
  });

  async function fetchMediaAppend() {
    try {
      if(currentType === 'videos') {
        const res = await fetch(`https://api.pexels.com/videos/search?query=${currentQuery}&per_page=${PER_PAGE}&page=${page}`, {
          headers: { Authorization: PEXELS_KEY }
        });
        const data = await res.json();
        grid.insertAdjacentHTML('beforeend', data.videos.map(v => {
          const file = v.video_files.find(f => f.quality === 'hd') || v.video_files[0];
          return `<div class="card"><video src="${file.link}" controls muted loop playsinline poster="${v.image}"></video><div class="card-body"><p>By: ${v.user.name}</p></div></div>`;
        }).join(''));
      }
    } finally {
      loadingMore = false;
    }
  }
</script>

</body>
</html>
