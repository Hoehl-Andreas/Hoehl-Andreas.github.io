<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Wave Tracker</title>
  <link rel="stylesheet" href="style.css">
  <link rel="icon" type="image/png" href="logo.png">
  <script src="https://cdnjs.cloudflare.com/ajax/libs/sql.js/1.8.0/sql-wasm.js"></script>
</head>
<body>
  <div class="container">
    <div class="header">
      <h1><img src="logo.png" alt="Wave Tracker Logo" class="header-logo"> Wave Tracker</h1>
      <p>Track It. Ride It. Own It.</p>
    </div>

    <div class="content">

      <!-- Video upload and detection section -->
      <div class="section">
        <h2 class="section-title">⬆️ Upload your video & detect waves</h2>
        <div class="controls" id="videoUploadArea">
          <div class="control-group">
            <label for="videoInput">Upload video:</label>
            <input id="videoInput" type="file" accept="video/*" />
            <button id="detectBtn" class="btn-primary">Detect Waves</button>
            <button id="clearVideoBtn" class="btn-secondary">Clear video</button>
          </div>
        </div>
        <div id="processingStatus"></div>
        <div id="videoContainer">
          <video id="videoPlayer" controls preload="metadata"></video>
        </div>
      </div>

      <!-- Filter and analysis section -->
      <div class="section">
        <h2 class="section-title">🔍 Filter & Analysis</h2>
        <div class="controls" id="filterArea">
          <div class="control-group">
            <label>Min duration (s):</label>
            <input id="minDuration" type="number" step="0.5" placeholder="0.00" />
          </div>
          <div class="control-group">
            <label>Max duration (s):</label>
            <input id="maxDuration" type="number" step="0.5" placeholder="10.00" />
          </div>
          <button id="loadBtn" class="btn-primary">Load data</button>
          <button id="clearBtn" class="btn-secondary">Clear filter</button>
          <button id="clearFramesBtn" class="btn-danger">Clear frames</button>
          <button id="clearDbBtn" class="btn-danger" style="background-color: #d32f2f;">Clear Database</button>
          <div id="count"></div>
        </div>
      </div>

      <!-- Results table section -->
      <div class="section">
        <h2 class="section-title">📊 Detection Results</h2>
        <table id="tbl">
          <thead>
                <tr>
                  <th>Wave ID</th>
                  <th>Tracking duration (s)</th>
                  <th>Video timestamp</th>
                  <th>Start Position (x,y)</th>
                  <th>Action</th>
                </tr>
          </thead>
          <tbody></tbody>
        </table>
      </div>
    </div>
  </div>

  <script>
    // Constrain the on-screen size for readability (overlay scales accordingly)
    const MAX_DISPLAY_WIDTH = 1140; // px (container 1200px - content padding)
    let tulips = [];
    let currentVideoURL = null;
    let videoWidth = 640;  // Default, will be updated from detector
    let videoHeight = 480;  // Default, will be updated from detector
    let lastShownPositions = []; // Store last shown positions for re-drawing

    // Video upload handlers
    const videoInput = document.getElementById('videoInput');
    const videoPlayer = document.getElementById('videoPlayer');
    const clearVideoBtn = document.getElementById('clearVideoBtn');

    videoInput.addEventListener('change', (e) => {
      const f = e.target.files && e.target.files[0];
      if (!f) return;
      if (currentVideoURL) {
        URL.revokeObjectURL(currentVideoURL);
        currentVideoURL = null;
      }
      currentVideoURL = URL.createObjectURL(f);
      videoPlayer.src = currentVideoURL;
      videoPlayer.load();
      // videoPlayer.play().catch(()=>{});
    });

    clearVideoBtn.addEventListener('click', () => {
      if (currentVideoURL) {
        URL.revokeObjectURL(currentVideoURL);
        currentVideoURL = null;
      }
      videoPlayer.removeAttribute('src');
      videoPlayer.load();
      videoInput.value = "";
    });

    document.getElementById('clearFramesBtn').addEventListener('click', () => {
      canvas.style.display = 'none';
      const ctx = canvas.getContext('2d');
      ctx.clearRect(0, 0, canvas.width, canvas.height);
    });

    // Status message helper
    function showStatus(message, type = 'loading') {
      const status = document.getElementById('processingStatus');
      status.textContent = message;
      status.className = type;
    }

    // Data load and rendering
    async function loadTulips() {
      showStatus('Loading data...', 'loading');
      
      // 1. Try to load directly from local wave_database.db file (Client-side / GitHub Pages mode)
      try {
        if (typeof initSqlJs === 'function') {
          const SQL = await initSqlJs({
            locateFile: file => `https://cdnjs.cloudflare.com/ajax/libs/sql.js/1.8.0/${file}`
          });
          
          const dbData = await fetch('wave_database.db').then(res => {
            if (!res.ok) throw new Error("Local database file not found");
            return res.arrayBuffer();
          });

          const db = new SQL.Database(new Uint8Array(dbData));
          
          // Attempt to find the main data table (ignoring system tables)
          let tableName = 'detections'; // Default guess
          try {
             // Try to find a table that looks like it holds our data
             const tables = db.exec("SELECT name FROM sqlite_master WHERE type='table' AND name NOT LIKE 'sqlite_%' AND name NOT LIKE 'video_%'");
             if (tables[0] && tables[0].values.length > 0) {
                 // Prefer 'detections' if it exists, otherwise take the first available
                 const names = tables[0].values.flat();
                 if (names.includes('detections')) {
                    tableName = 'detections';
                 } else {
                    tableName = names[0];
                 }
                 console.log(`Found tables: ${names.join(', ')}. Using: ${tableName}`);
             }
          } catch(e) { console.log("Using default table name 'detections'"); }

          // Select columns based on typical schema
          const stmt = db.prepare(`SELECT * FROM ${tableName}`);
          tulips = [];
          
          while (stmt.step()) {
            const row = stmt.getAsObject();
            // Map common DB column names to our expected format
            tulips.push({
              id: row.id || row.wave_id || row.object_id,
              duration: row.duration || 0,
              first_detected_at: row.start_time || row.timestamp || row.first_detected_at || 0,
              position: {
                x: row.start_x || row.x || 0,
                y: row.start_y || row.y || 0
              }
            });
          }
          stmt.free();
          db.close();
          
          render(tulips);
          
          // CRITICAL FIX: Initialize display size so canvas exists even if no video file is uploaded yet
          // Assuming 720p default aspect ratio if unknown
          if (videoPlayer.videoWidth) {
              videoWidth = videoPlayer.videoWidth;
              videoHeight = videoPlayer.videoHeight;
          }
          setVideoDisplaySize(videoWidth, videoHeight);
          
          showStatus('Data loaded from wave_database.db', 'success');
          setTimeout(() => { document.getElementById('processingStatus').style.display = 'none'; }, 2000);
          return; // Exit if successful
        }
      } catch (dbErr) {
        console.log("Could not load local DB file, trying API fallback...", dbErr);
      }
      
      // 2. Fallback: Try API (for backend usage)
      try {
        // Ensure correct scaling before drawing points
        try {
          const dimsRes = await fetch('/api/video_dims');
          if (dimsRes.ok) {
            const dims = await dimsRes.json();
            if (dims && dims.processed_width && dims.processed_height) {
              videoWidth = dims.processed_width;
              videoHeight = dims.processed_height;
              setVideoDisplaySize(videoWidth, videoHeight);
              console.log(`Using DB processed dims: ${videoWidth}x${videoHeight}`);
            }
          }
        } catch (e) {
          console.warn('Failed to load processed dimensions', e);
        }

        const res = await fetch("/api/tulips");
        if (!res.ok) throw new Error("HTTP " + res.status);
        tulips = await res.json();
        render(tulips);
        showStatus('Data loaded successfully', 'success');
        setTimeout(() => {
          document.getElementById('processingStatus').style.display = 'none';
        }, 3000);
      } catch (err) {
        showStatus('Error loading data: ' + err.message, 'error');
      }
    }

    function formatTimeHMS(seconds) {
      if (typeof seconds !== 'number' || !isFinite(seconds)) return '-';
      const t = Math.floor(seconds);
      const h = Math.floor(t / 3600);
      const m = Math.floor((t % 3600) / 60);
      const s = t % 60;
      const hh = String(h).padStart(2, '0');
      const mm = String(m).padStart(2, '0');
      const ss = String(s).padStart(2, '0');
      return `${hh}:${mm}:${ss}`;
    }

    function render(list) {
      const tbody = document.querySelector("#tbl tbody");
      tbody.innerHTML = "";
      for (const t of list) {
        const tr = document.createElement("tr");
        const firstTime = formatTimeHMS(t.first_detected_at);
        tr.innerHTML = `
            <td>${escapeHtml(t.id)}</td>
            <td>${t.duration.toFixed(2)}</td>
            <td>${firstTime}</td>
            <td>(${t.position.x}, ${t.position.y})</td>
            <td><button class="btn-secondary" onclick="showPositions([{id: '${escapeHtml(t.id)}', x: ${t.position.x}, y: ${t.position.y}}])" style="padding: 6px 12px; font-size: 0.9em;">Show</button></td>
        `;
        tbody.appendChild(tr);
      }
      
      // Remove existing "Show All Filtered" button if present
      const existingBtn = document.getElementById("showAllFilteredBtn");
      if (existingBtn) {
          existingBtn.remove();
      }
      
      // Add "Show All" button if there are entries
      if (list.length > 0) {
          const showAllBtn = document.createElement("button");
          showAllBtn.id = "showAllFilteredBtn";
          showAllBtn.className = "btn-primary";
          showAllBtn.textContent = "Show filtered IDs";
          showAllBtn.style.marginLeft = "10px";
          showAllBtn.onclick = () => showPositions(
              list.map(t => ({
                  id: t.id,
                  x: t.position.x,
                  y: t.position.y
              }))
          );
          document.getElementById("filterArea").appendChild(showAllBtn);
      }
      
      document.getElementById("count").textContent = 
          `Showing ${list.length} of ${tulips.length} results`;
    }

    function escapeHtml(s){ return String(s).replace(/[&<>"']/g, c=>({ '&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;' }[c])); }

    function applyFilter() {
      const min = parseFloat(document.getElementById("minDuration").value);
      const max = parseFloat(document.getElementById("maxDuration").value);
      const hasMin = !Number.isNaN(min);
      const hasMax = !Number.isNaN(max);
      const filtered = tulips.filter(item => {
        const duration = Number(item.duration || 0);
        if (!isFinite(duration)) return false;
        if (hasMin && duration < min) return false;
        if (hasMax && duration > max) return false;
        return true;
      });
      render(filtered);
    }

    document.getElementById("minDuration").addEventListener("input", applyFilter);
    document.getElementById("maxDuration").addEventListener("input", applyFilter);
    document.getElementById("clearBtn").addEventListener("click", () => {
      document.getElementById("minDuration").value = "";
      document.getElementById("maxDuration").value = "";
      render(tulips);
    });
    document.getElementById("loadBtn").addEventListener("click", loadTulips);
    

    document.getElementById("clearDbBtn").addEventListener("click", async () => {
        if (!confirm("Are you sure you want to clear all detection data from the database?")) return;
        try {
            const res = await fetch("/api/clear_db", { method: "POST" });
            if (res.ok) {
                tulips = [];
                render([]);
                showStatus('Database cleared', 'success');
                setTimeout(() => document.getElementById('processingStatus').style.display = 'none', 2000);
            } else {
                showStatus('Failed to clear database', 'error');
            }
        } catch (e) {
            showStatus('Error: ' + e.message, 'error');
        }
    });

    // Object detection
    const detectBtn = document.getElementById('detectBtn');
    const processingStatus = document.getElementById('processingStatus');

    detectBtn.addEventListener('click', async () => {
    const videoFile = videoInput.files[0];
    console.log('Detect button clicked');
    
    if (!videoFile) {
        console.log('No file selected');
        showStatus('Please select a video first', 'error');
        return;
    }

    detectBtn.disabled = true;
    showStatus('⏳ Processing video... this will take a view minutes', 'loading');
    console.log('Sending file:', videoFile.name);

    const formData = new FormData();
    formData.append('video', videoFile);

    try {
        console.log('Sending request to /detect');
        const response = await fetch('/detect', {
            method: 'POST',
            body: formData
        });

        console.log('Response status:', response.status);
        const result = await response.json();
        console.log('Response data:', result);

        if (!response.ok) {
            throw new Error(result.error || 'Processing failed');
        }

        // Keep the original uploaded video in the player
        if (result.input_url && !currentVideoURL) {
          // If for some reason the local URL was lost, use the server URL
          videoPlayer.src = result.input_url;
        }
        // Otherwise keep the current blob URL showing the original video

        // Ensure metadata is ready before sizing/overlay
        await ensureVideoReady(videoPlayer);
        updateCanvasSize();

        // Use dimensions from detector result if available, as coordinates are relative to that
        // This matches the old version behavior where we trust the detector's coordinate system
        if (result.video_width && result.video_height) {
            videoWidth = result.video_width;
            videoHeight = result.video_height;
            console.log(`Using detector dimensions: ${videoWidth}x${videoHeight}`);
        } else {
            videoWidth = videoPlayer.videoWidth || 640;
            videoHeight = videoPlayer.videoHeight || 480;
            console.log(`Using video player dimensions: ${videoWidth}x${videoHeight}`);
        }
        
        console.log(`Video element display size: ${videoPlayer.offsetWidth}x${videoPlayer.offsetHeight}`);
        
        showStatus(`✅ Detection complete. Found ${result.objects_count} objects.`, 'success');

        // Load data from database and display red dots with IDs
        await loadAndDisplayDetections();

    } catch (error) {
        console.error('Detection error:', error);
        showStatus(`❌ Error: ${error.message}`, 'error');
    } finally {
        detectBtn.disabled = false;
    }
});

// New function to load detection data from database and display overlay
async function loadAndDisplayDetections() {
  try {
    const res = await fetch("/api/tulips");
    if (!res.ok) throw new Error("HTTP " + res.status);
    tulips = await res.json();
    // Ensure we use the correct processed dimensions before drawing
    try {
      const dimsRes = await fetch('/api/video_dims');
      if (dimsRes.ok) {
        const dims = await dimsRes.json();
        if (dims && dims.processed_width && dims.processed_height) {
          videoWidth = dims.processed_width;
          videoHeight = dims.processed_height;
          console.log(`Loaded processed dims from DB: ${videoWidth}x${videoHeight}`);
          setVideoDisplaySize(videoWidth, videoHeight);
        }
      }
    } catch (e) {
      console.warn('Failed to load processed dimensions, using existing values.', e);
    }
    
    // Display red dots for all detections from database
    if (tulips.length > 0) {
      const positions = tulips.map(t => ({
        id: t.id,
        x: t.position.x,
        y: t.position.y
      }));
      await showPositions(positions);
    }
    
    // Update the results table
    render(tulips);
  } catch (err) {
    console.error('Error loading detections:', err);
  }
}

// Add canvas overlay for showing positions
const canvas = document.createElement('canvas');
canvas.style.position = 'absolute';
canvas.style.pointerEvents = 'none';
canvas.style.display = 'block';
canvas.style.top = '0';
canvas.style.left = '0';
canvas.style.zIndex = '10';
// Ensure the overlay stacks correctly above the video
const videoContainerEl = document.getElementById('videoContainer');
if (videoContainerEl) {
  videoContainerEl.style.position = 'relative';
}
document.getElementById('videoContainer').appendChild(canvas);

// Add green reference dots in top-left and bottom-right corners
function drawCornerReferenceDots() {
  const video = document.getElementById('videoPlayer');
  if (!video.offsetWidth || !video.offsetHeight) return;

  const ctx = canvas.getContext('2d');
  const videoDisplayWidth = video.offsetWidth;
  const videoDisplayHeight = video.offsetHeight;

  const scaleX = videoDisplayWidth / videoWidth;
  const scaleY = videoDisplayHeight / videoHeight;

  const points = [
    { x: 0, y: 0, label: `(0, 0)`, align: 'left', baseline: 'top', dx: 8, dy: 8 },
    { x: videoWidth, y: videoHeight, label: `(${videoWidth}, ${videoHeight})`, align: 'right', baseline: 'bottom', dx: -8, dy: -8 },
  ];

  ctx.save();
  ctx.fillStyle = 'limegreen';
  ctx.strokeStyle = 'limegreen';
  ctx.font = '12px Arial';

  points.forEach(p => {
    const px = p.x * scaleX;
    const py = p.y * scaleY;

    // Dot
    ctx.beginPath();
    ctx.arc(px, py, 6, 0, 2 * Math.PI);
    ctx.fill();

    // Label
    ctx.textAlign = p.align;
    ctx.textBaseline = p.baseline;
    ctx.fillText(p.label, px + p.dx, py + p.dy);
  });

  ctx.restore();
}


// Draw a permanent blue dot at fixed coordinates
function drawBlueDot() {
  // This function previously cleared the canvas via updateCanvasSize()
  // We removed that call to prevent wiping out our red dots
  // It is now strictly a placeholder or safe update
  const video = document.getElementById('videoPlayer');
  if (!video.offsetWidth) return;
  // No-op to avoid clearing detection markers
}

async function showPositions(positions) {
        const video = document.getElementById('videoPlayer');
        await ensureVideoReady(video);
        
        // FORCE update dimensions based on current video state
        if (video.videoWidth) {
            videoWidth = video.videoWidth;
            videoHeight = video.videoHeight;
        }
        
        // Ensure canvas size matches video display size
        const w = video.offsetWidth || 640;
        const h = video.offsetHeight || 480;
        canvas.width = w;
        canvas.height = h;
        canvas.style.display = 'block';
        
        const ctx = canvas.getContext('2d');
        // Explicitly clear before drawing
        ctx.clearRect(0, 0, canvas.width, canvas.height);
    
        const videoDisplayWidth = canvas.width;
        const videoDisplayHeight = canvas.height;
        
        // Scale factors: from detector dimensions to displayed canvas dimensions
        const scaleX = videoDisplayWidth / videoWidth;
        const scaleY = videoDisplayHeight / videoHeight;
        
        console.log(`Drawing ${positions.length} markers. Scale: ${scaleX.toFixed(3)}`);
        lastShownPositions = positions;
    
        positions.forEach(({id, x, y}) => {
            const scaledX = x * scaleX;
            const scaledY = y * scaleY;
            
            // Draw red dot
            ctx.beginPath();
            ctx.arc(scaledX, scaledY, 6, 0, 2 * Math.PI); // Slightly larger
            ctx.fillStyle = '#ff0000';
            ctx.fill();
            ctx.lineWidth = 2;
            ctx.strokeStyle = 'white';
            ctx.stroke();
            
            // Draw ID label
            ctx.font = 'bold 14px Arial';
            ctx.fillStyle = '#ff0000';
            ctx.fillText(id, scaledX + 12, scaledY + 4);
        });
    }

// Clear markers when video changes
videoInput.addEventListener('change', () => {
    updateCanvasSize();
    const ctx = canvas.getContext('2d');
    ctx.clearRect(0, 0, canvas.width, canvas.height);
    drawBlueDot();
});

// Update canvas size when video size changes
window.addEventListener('resize', () => { updateCanvasSize(); drawBlueDot(); });
videoPlayer.addEventListener('loadedmetadata', () => {
  // CRITICAL FIX: Update global reference dimensions to match the uploaded video
  // This ensures the DB coordinates (which match this video) scale correctly to the display
  if (videoPlayer.videoWidth && videoPlayer.videoHeight) {
      videoWidth = videoPlayer.videoWidth;
      videoHeight = videoPlayer.videoHeight;
      console.log(`Updated reference dimensions from video: ${videoWidth}x${videoHeight}`);
  }

  // Default to the video's intrinsic resolution when metadata is ready
  const w = videoPlayer.videoWidth || videoWidth;
  const h = videoPlayer.videoHeight || videoHeight;
  setVideoDisplaySize(w, h);
  drawBlueDot();
});

// Add: initialize processed dims on page load
document.addEventListener('DOMContentLoaded', async () => {
  try {
    const dimsRes = await fetch('/api/video_dims');
    if (dimsRes.ok) {
      const dims = await dimsRes.json();
      if (dims && dims.processed_width && dims.processed_height) {
        videoWidth = dims.processed_width;
        videoHeight = dims.processed_height;
        setVideoDisplaySize(videoWidth, videoHeight);
        console.log(`Init processed dims: ${videoWidth}x${videoHeight}`);
      }
    }
  } catch (e) {
    console.warn('Failed to init processed dims', e);
  }
});
  </script>
</body>
</html>
