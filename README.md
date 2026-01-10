
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Wave Tracker</title>
  <link rel="stylesheet" href="style.css">
  <link rel="icon" type="image/png" href="logo.png">
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
  const video = document.getElementById('videoPlayer');
  // Guard against drawing on a zero-sized canvas before metadata is ready
  if (!video.offsetWidth || !video.offsetHeight) return;
  updateCanvasSize();
  const ctx = canvas.getContext('2d');
  const videoDisplayWidth = video.offsetWidth;
  const videoDisplayHeight = video.offsetHeight;
  // Scale the fixed coordinates from detector space to display space
  const scaleX = videoDisplayWidth / videoWidth;
  const scaleY = videoDisplayHeight / videoHeight;
  // Blue dot removed per request; this function is now a no-op

}

// Draw initial blue dot on load
updateCanvasSize();
drawBlueDot();

function updateCanvasSize() {
    const video = document.getElementById('videoPlayer');
    canvas.width = video.offsetWidth;
    canvas.height = video.offsetHeight;
    // Revert to simple positioning like the old version
    canvas.style.top = '0';
    canvas.style.left = '0';
}

// Ensure the video element and its container match the detector's coordinate space
function setVideoDisplaySize(w, h) {
  const video = document.getElementById('videoPlayer');
  const container = document.getElementById('videoContainer');
  if (!w || !h) return;
  // Fill available width within the content area, capped at MAX_DISPLAY_WIDTH
  const parent = container.parentElement || document.body;
  const parentWidth = parent.clientWidth || window.innerWidth;
  const targetW = Math.min(parentWidth, MAX_DISPLAY_WIDTH);
  const targetH = Math.round(targetW * (h / w));
  video.style.width = targetW + 'px';
  video.style.height = targetH + 'px';
  container.style.width = targetW + 'px';
  container.style.height = targetH + 'px';
  updateCanvasSize();
}

function ensureVideoReady(video) {
  return new Promise(resolve => {
    if (video.readyState >= 1 && video.videoWidth && video.videoHeight) {
      resolve();
    } else {
      const onMeta = () => { video.removeEventListener('loadedmetadata', onMeta); resolve(); };
      video.addEventListener('loadedmetadata', onMeta);
    }
  });
}

async function showPositions(positions) {
    const video = document.getElementById('videoPlayer');
    await ensureVideoReady(video);
    updateCanvasSize();
    canvas.style.display = 'block';
    
    const ctx = canvas.getContext('2d');
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    // Get the actual video display dimensions
    const videoDisplayWidth = video.offsetWidth;
    const videoDisplayHeight = video.offsetHeight;
    
    // Scale factors: from detector dimensions (1280px) to displayed canvas dimensions
    const scaleX = videoDisplayWidth / videoWidth;
    const scaleY = videoDisplayHeight / videoHeight;
    
    console.log('=== COORDINATE DEBUGGING ===');
    console.log(`Detector dimensions: ${videoWidth}x${videoHeight}`);
    console.log(`Display dimensions: ${videoDisplayWidth}x${videoDisplayHeight}`);
    console.log(`Scale factors: ${scaleX.toFixed(4)}, ${scaleY.toFixed(4)}`);
    lastShownPositions = positions;

    // Draw permanent blue dot at fixed coords
    drawBlueDot();

    positions.forEach(({id, x, y}) => {
        const scaledX = x * scaleX;
    const scaledY = y * scaleY;
        
        // Draw red dot
        ctx.beginPath();
        ctx.arc(scaledX, scaledY, 5, 0, 2 * Math.PI);
        ctx.fillStyle = 'red';
        ctx.fill();
        
        // Draw ID label
        ctx.font = '12px Arial';
        ctx.fillStyle = 'red';
        ctx.fillText(id, scaledX + 10, scaledY - 5);
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
