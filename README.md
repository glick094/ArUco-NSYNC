# ArUco NSYNC

A real-time computer vision synchronization tool that displays high-precision timestamps and dynamic ArUco-style markers for multi-camera calibration and synchronization.

## Features

- **Millisecond-precision Unix timestamps** for accurate temporal synchronization
- **Dynamic ArUco-style markers** that change based on current time (day, hour, minute, second, millisecond)
- **Professional clapper/sync button** with standardized audio-visual sequence
- **Keyboard activation** - Press Enter to trigger clapper from anywhere
- **Automatic CSV export** - Programmatic sync data for post-processing
- **Two-stage sync system**: Coarse (1.5s) + high-precision (200ms) sequences
- **High-contrast display** optimized for computer vision detection
- **Screen resolution and calibration data** for camera setup
- **Geometric font rendering** for additional feature point detection
- **Responsive layout** ensures all markers visible without scrolling
- **Fullscreen support** for maximum marker visibility

## Live Demo

Access the app at: `https://YOUR_USERNAME.github.io/aruco-nsync/`

## Technical Implementation

### 1. Unix Timestamp Generation

The app generates millisecond-precision timestamps using JavaScript's native `Date.getTime()` method:

```javascript
function updateDisplay() {
    const now = new Date();
    const timestamp = now.getTime(); // Milliseconds since Unix epoch
    
    document.getElementById('timestamp').textContent = `Unix: ${timestamp}`;
    // ... rest of function
}
```

**Key Points:**
- Updates every 10ms via `setInterval(updateDisplay, 10)`
- Provides millisecond precision (e.g., `1692547234567`)
- High-frequency updates for precise frame synchronization
- Updates ~1.7 times per 60fps frame (60fps = 16.67ms per frame)

### 2. ArUco Marker Generation

Dynamic markers are generated using binary representation of time values:

```javascript
function generateMarkerPattern(value) {
    const binary = value.toString(2).padStart(25, '0');
    const pattern = Array(7).fill().map(() => Array(7).fill(1));
    
    // Fill inner 5x5 area with binary pattern
    for (let i = 0; i < 5; i++) {
        for (let j = 0; j < 5; j++) {
            pattern[i + 1][j + 1] = parseInt(binary[i * 5 + j]);
        }
    }
    return pattern;
}

function renderMarker(containerId, pattern) {
    const container = document.getElementById(containerId);
    container.innerHTML = '';
    
    pattern.flat().forEach(cell => {
        const div = document.createElement('div');
        div.className = `marker-cell ${cell ? 'black' : 'white'}`;
        container.appendChild(div);
    });
}
```

**Marker Specifications:**
- **Format:** 7×7 grid with black border, 5×5 inner data area
- **Encoding:** 25-bit binary representation of time values
- **Sizes:** Main markers (400×400px), Corner marker (120×120px)
- **Update Rate:** Every second with new time-based patterns

**Time-to-Marker Mapping:**
```javascript
const dayOfYear = Math.floor((now - new Date(now.getFullYear(), 0, 0)) / (1000 * 60 * 60 * 24));
const hour = now.getHours();        // 0-23
const minute = now.getMinutes();    // 0-59  
const second = now.getSeconds();    // 0-59

renderMarker('dayMarker', generateMarkerPattern(dayOfYear));     // 1-365
renderMarker('hourMarker', generateMarkerPattern(hour));        // 0-23
renderMarker('minuteMarker', generateMarkerPattern(minute));    // 0-59
renderMarker('secondMarker', generateMarkerPattern(second));    // 0-59
```

### 3. Precision and Uncertainty Analysis

#### Timing Precision
- **JavaScript Timer Resolution:** ~1-4ms depending on browser and system
- **Display Update Rate:** 10ms (100Hz refresh rate)
- **Frame Alignment Precision:** ~0.6 updates per 60fps frame (16.67ms)
- **Network Synchronization:** ±50-200ms typical variance across devices
- **Sub-frame Timing:** Can capture timestamp changes between video frames

#### Spatial Precision
- **Marker Cell Size:** 400px ÷ 7 = ~57px per cell (main markers)
- **Pixel Accuracy:** ±0.5 pixels for well-lit, high-contrast detection
- **Distance Factors:** Precision degrades with camera distance and angle

#### Enhanced Synchronization Benefits
```javascript
// With 10ms updates, you get precise frame-level sync
const UPDATE_INTERVAL_MS = 10;        // 100Hz update rate
const FRAME_INTERVAL_60FPS = 16.67;   // 60fps = 16.67ms per frame
const UPDATES_PER_FRAME = FRAME_INTERVAL_60FPS / UPDATE_INTERVAL_MS; // ~1.7 updates per frame

// This allows sub-frame timestamp precision
const SYNC_TOLERANCE_MS = 10;   // Much tighter tolerance possible
const FRAME_BUFFER_SIZE = 1;    // Single frame precision achievable
```

#### Recommended Setup Tolerances
```javascript
// Timing tolerances for high-precision multi-camera sync
const SYNC_TOLERANCE_MS = 10;    // ±10ms variance (was 100ms)
const FRAME_BUFFER_SIZE = 1;     // Single frame precision
const TEMPORAL_RESOLUTION_MS = 10; // 100Hz temporal sampling

// Spatial tolerances for marker detection  
const MIN_MARKER_SIZE_PIXELS = 50;   // Minimum detectable marker size
const MAX_CAMERA_ANGLE_DEG = 45;     // Maximum viewing angle for reliable detection
```

### 4. Multi-Camera Synchronization and Triangulation

#### Setup Process

1. **Deploy on Network-Accessible Device**
   ```bash
   # Option 1: Local server
   python -m http.server 8000
   # Access at http://YOUR_IP:8000
   
   # Option 2: GitHub Pages (recommended)
   # Access at https://username.github.io/aruco-nsync/
   ```

2. **Camera Positioning**
   - Place display device in center of capture volume
   - Position cameras with clear view of all markers
   - Ensure minimum 50px marker size in camera view
   - Maintain viewing angles < 45° for optimal detection

#### Synchronization Workflow

```python
# Example OpenCV detection code
import cv2
import time
import numpy as np

def detect_sync_markers(frame, timestamp_ms):
    """
    Detect ArUco-style markers and extract timing information
    """
    # Convert to grayscale
    gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
    
    # Detect contours for marker regions
    contours, _ = cv2.findContours(gray, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE)
    
    # Filter for square-like contours (markers)
    markers = []
    for contour in contours:
        # Check if contour is roughly square and large enough
        area = cv2.contourArea(contour)
        if area > 2500:  # Minimum marker size
            # Extract marker pattern and decode time value
            marker_value = decode_marker_pattern(contour, gray)
            markers.append({
                'position': cv2.boundingRect(contour),
                'value': marker_value,
                'timestamp': timestamp_ms,
                'camera_id': get_camera_id()
            })
    
    return markers

def synchronize_cameras(camera_feeds):
    """
    Synchronize multiple camera feeds using high-frequency marker timestamps
    """
    sync_data = {}
    
    for camera_id, feed in camera_feeds.items():
        frame = feed.read()
        capture_time = time.time() * 1000  # Convert to milliseconds
        
        markers = detect_sync_markers(frame, capture_time)
        
        # With 10ms updates, can achieve sub-frame synchronization
        # Look for the most recent timestamp update
        latest_timestamp = find_latest_timestamp_marker(markers)
        if latest_timestamp:
            sync_data[camera_id] = {
                'frame': frame,
                'display_time': latest_timestamp['timestamp'],
                'capture_time': capture_time,
                'sync_precision': abs(capture_time - latest_timestamp['timestamp']),
                'markers': markers
            }
    
    return align_frames_by_high_precision_sync(sync_data)
```

#### Triangulation Applications

```python
def triangulate_with_markers(sync_data):
    """
    Use detected markers for 3D triangulation
    """
    # Extract marker positions from synchronized frames
    marker_positions_2d = {}
    
    for camera_id, data in sync_data.items():
        positions = []
        for marker in data['markers']:
            # Get center point of each marker
            x, y, w, h = marker['position']
            center = (x + w//2, y + h//2)
            positions.append({
                'type': marker['type'],  # hour, minute, second, day
                'position_2d': center,
                'camera_id': camera_id
            })
        marker_positions_2d[camera_id] = positions
    
    # Perform triangulation using camera calibration matrices
    triangulated_points = []
    for marker_type in ['hour', 'minute', 'second', 'day']:
        points_2d = get_marker_positions_by_type(marker_positions_2d, marker_type)
        if len(points_2d) >= 2:  # Need at least 2 cameras
            point_3d = cv2.triangulatePoints(
                camera_matrices[0], camera_matrices[1],
                points_2d[0], points_2d[1]
            )
            triangulated_points.append({
                'type': marker_type,
                'position_3d': point_3d,
                'timestamp': sync_data[list(sync_data.keys())[0]]['display_time']
            })
    
    return triangulated_points
```

#### Best Practices

1. **High-Precision Timing Synchronization**
   - Capture frames within ±10ms for sub-frame precision
   - Use millisecond timestamp changes for frame alignment
   - No need for frame buffering - single frame precision achievable
   - Monitor for timestamp discontinuities during capture

2. **Spatial Calibration**
   - Use hour/minute/day markers as static reference points during setup
   - Calibrate cameras using marker corner detection
   - Account for display screen physical dimensions
   - Validate marker detection at target frame rates

3. **Quality Assurance**
   - Verify marker detection confidence scores
   - Cross-validate synchronization precision across all cameras
   - Monitor timing drift over extended capture sessions
   - Test synchronization accuracy at various frame rates (30fps, 60fps, 120fps)

4. **Frame Rate Optimization**
   ```python
   # Optimal sync windows for different frame rates
   FRAME_RATES = {
       30: 33.33,   # 30fps = 33.33ms per frame (3+ timestamp updates per frame)
       60: 16.67,   # 60fps = 16.67ms per frame (1.7 timestamp updates per frame)  
       120: 8.33    # 120fps = 8.33ms per frame (0.8 timestamp updates per frame)
   }
   
   # For 120fps, capture multiple frames per timestamp update
   # For 30/60fps, get multiple timestamp updates per frame
   ```

## Browser Compatibility & Performance

- **Chrome/Edge:** Full support, ~1ms timing precision, 10ms updates
- **Firefox:** Full support, ~1ms timing precision, 10ms updates  
- **Safari:** Full support, ~4ms timing precision, 10ms updates
- **Mobile browsers:** Supported, may throttle to 16ms+ updates when backgrounded

**Performance Notes:**
- 10ms update rate provides excellent sub-frame precision for most video applications
- For 60fps video (16.67ms/frame): ~1.7 timestamp updates per frame
- For 30fps video (33.33ms/frame): ~3.3 timestamp updates per frame
- For 120fps video (8.33ms/frame): timestamp updates every ~1.2 frames

## Hardware Requirements

- **Display:** Any web-capable device with screen
- **Network:** Local network or internet connection
- **Cameras:** Computer vision setup with OpenCV or similar
- **Lighting:** Adequate lighting for high-contrast marker detection

## Contributing

Feel free to submit issues or pull requests to improve marker generation, timing precision, or add new features for computer vision applications.

## License

MIT License - feel free to use in your computer vision projects.