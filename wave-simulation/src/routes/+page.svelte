<script>
  import { onMount, onDestroy } from 'svelte';

  // --- Configuration ---
  let canvasWidth = 0;
  let canvasHeight = 0;
  let amplitude = 1;
  let dotDensityFactor = 0.1;
  let decayFactor = 0;
  const sourceRadius = 6;
  const dragSensitivity = sourceRadius * 2.5;

  // --- Source Management ---
  let sources = [];

  // --- Canvas & Simulation State ---
  let webglCanvas;
  let overlayCanvas;
  let gl;
  let program;
  let animationFrameId;
  let startTime = 0;
  let currentTime = 0;

  // --- Dragging State ---
  let draggingSourceId = null;
  let dragOffsetX = 0;
  let dragOffsetY = 0;
  let isDragging = false;

  // --- Derived Wave Parameters ---
  $: maxPossibleAmplitude = sources.length * amplitude;

  // --- WebGL Setup ---
  const vertexShaderSource = `
    attribute vec2 a_position;
    void main() {
      gl_Position = vec4(a_position, 0.0, 1.0);
    }
  `;

  const fragmentShaderSource = `
    precision highp float;
    uniform vec2 u_resolution;
    uniform float u_time;
    uniform float u_amplitude;
    uniform float u_dotDensityFactor;
    uniform float u_decayFactor;
    uniform int u_numSources;
    uniform vec2 u_sourcePositions[8];
    uniform float u_wavelengths[8];
    uniform float u_frequencies[8];

    float random(vec2 st) {
      return fract(sin(dot(st, vec2(12.9898, 78.233))) * 43758.5453123);
    }

    void main() {
      vec2 pixelPos = gl_FragCoord.xy;
      float totalValue = 0.0;
      float maxAmplitude = float(u_numSources) * u_amplitude;

      for (int i = 0; i < 8; i++) {
        if (i >= u_numSources) break;
        vec2 sourcePos = u_sourcePositions[i];
        float dist = distance(pixelPos, sourcePos);
        if (dist < 0.001) dist = 0.001;
        float k = 2.0 * 3.14159265359 / u_wavelengths[i];
        float omega = 2.0 * 3.14159265359 * u_frequencies[i];
        float amplitude = u_amplitude;
        if (u_decayFactor > 0.0 && dist > 0.0) {
          amplitude /= pow(dist, u_decayFactor);
        }
        float wave = amplitude * cos(k * dist - omega * u_time);
        totalValue += wave;
      }

      float normalizedValue = (totalValue + maxAmplitude) / (2.0 * maxAmplitude);
      normalizedValue = clamp(normalizedValue, 0.0, 1.0);
      float probability = normalizedValue * u_dotDensityFactor;
      probability = clamp(probability, 0.0, 1.0);

      vec3 color = vec3(0.0);
      if (random(gl_FragCoord.xy + vec2(u_time)) < probability) {
        color = vec3(1.0);
      }

      gl_FragColor = vec4(color, 1.0);
    }
  `;

  function createShader(gl, type, source) {
    const shader = gl.createShader(type);
    gl.shaderSource(shader, source);
    gl.compileShader(shader);
    if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
      console.error(`Shader compile error (${type === gl.VERTEX_SHADER ? 'vertex' : 'fragment'}):`, gl.getShaderInfoLog(shader));
      gl.deleteShader(shader);
      return null;
    }
    return shader;
  }

  function createProgram(gl, vertexSource, fragmentSource) {
    const vertexShader = createShader(gl, gl.VERTEX_SHADER, vertexSource);
    const fragmentShader = createShader(gl, gl.FRAGMENT_SHADER, fragmentSource);
    if (!vertexShader || !fragmentShader) return null;
    const program = gl.createProgram();
    gl.attachShader(program, vertexShader);
    gl.attachShader(program, fragmentShader);
    gl.linkProgram(program);
    if (!gl.getProgramParameter(program, gl.LINK_STATUS)) {
      console.error('Program link error:', gl.getProgramInfoLog(program));
      gl.deleteProgram(program);
      return null;
    }
    return program;
  }

  function initWebGL() {
    gl = webglCanvas.getContext('webgl', { preserveDrawingBuffer: true });
    if (!gl) {
      console.error('WebGL not supported or context creation failed');
      alert('WebGL is not supported in your browser.');
      return false;
    }

    program = createProgram(gl, vertexShaderSource, fragmentShaderSource);
    if (!program) {
      console.error('Failed to create WebGL program');
      return false;
    }

    const vertices = new Float32Array([-1, -1, 1, -1, -1, 1, 1, 1]);
    const vertexBuffer = gl.createBuffer();
    gl.bindBuffer(gl.ARRAY_BUFFER, vertexBuffer);
    gl.bufferData(gl.ARRAY_BUFFER, vertices, gl.STATIC_DRAW);

    const positionLocation = gl.getAttribLocation(program, 'a_position');
    gl.enableVertexAttribArray(positionLocation);
    gl.vertexAttribPointer(positionLocation, 2, gl.FLOAT, false, 0, 0);

    gl.useProgram(program);
    return true;
  }

  onMount(() => {
    if (!initWebGL()) return;

    sources = [
      { id: 1, x: canvasWidth * 0.3, y: canvasHeight / 2, color: 'rgba(255, 100, 100, 0.9)', wavelength: 100, frequency: 1.0 },
      { id: 2, x: canvasWidth * 0.7, y: canvasHeight / 2, color: 'rgba(100, 100, 255, 0.9)', wavelength: 100, frequency: 1.0 }
    ];

    startTime = performance.now();

    overlayCanvas.addEventListener('mousedown', handleMouseDown);
    window.addEventListener('mousemove', handleMouseMove);
    window.addEventListener('mouseup', handleMouseUp);

    overlayCanvas.addEventListener('touchstart', handleTouchStart, { passive: false });
    window.addEventListener('touchmove', handleTouchMove, { passive: false });
    window.addEventListener('touchend', handleTouchEnd);
    window.addEventListener('touchcancel', handleTouchEnd);

    animationLoop();

    return () => {
      if (animationFrameId) cancelAnimationFrame(animationFrameId);
      overlayCanvas?.removeEventListener('mousedown', handleMouseDown);
      window.removeEventListener('mousemove', handleMouseMove);
      window.removeEventListener('mouseup', handleMouseUp);
      overlayCanvas?.removeEventListener('touchstart', handleTouchStart);
      window.removeEventListener('touchmove', handleTouchMove);
      window.removeEventListener('touchend', handleTouchEnd);
      window.removeEventListener('touchcancel', handleTouchEnd);
      if (gl && program) gl.deleteProgram(program);
    };
  });

  function distance(x1, y1, x2, y2) {
    return Math.sqrt((x1 - x2) ** 2 + (y1 - y2) ** 2);
  }

  function drawVisualization(time) {
    if (!gl || !program) return;

    gl.viewport(0, 0, canvasWidth, canvasHeight);
    gl.clearColor(0, 0, 0, 1);
    gl.clear(gl.COLOR_BUFFER_BIT);

    const resolutionLocation = gl.getUniformLocation(program, 'u_resolution');
    gl.uniform2f(resolutionLocation, canvasWidth, canvasHeight);

    const timeLocation = gl.getUniformLocation(program, 'u_time');
    gl.uniform1f(timeLocation, time);

    const amplitudeLocation = gl.getUniformLocation(program, 'u_amplitude');
    gl.uniform1f(amplitudeLocation, amplitude);

    const dotDensityLocation = gl.getUniformLocation(program, 'u_dotDensityFactor');
    gl.uniform1f(dotDensityLocation, dotDensityFactor);

    const decayFactorLocation = gl.getUniformLocation(program, 'u_decayFactor');
    gl.uniform1f(decayFactorLocation, decayFactor);

    const numSourcesLocation = gl.getUniformLocation(program, 'u_numSources');
    gl.uniform1i(numSourcesLocation, sources.length);

    const sourcePositions = new Float32Array(sources.length * 2);
    const wavelengths = new Float32Array(sources.length);
    const frequencies = new Float32Array(sources.length);

    sources.forEach((source, i) => {
      sourcePositions[i * 2] = source.x;
      sourcePositions[i * 2 + 1] = canvasHeight - source.y;
      wavelengths[i] = source.wavelength;
      frequencies[i] = source.frequency;
    });

    const sourcePositionsLocation = gl.getUniformLocation(program, 'u_sourcePositions');
    gl.uniform2fv(sourcePositionsLocation, sourcePositions);

    const wavelengthsLocation = gl.getUniformLocation(program, 'u_wavelengths');
    gl.uniform1fv(wavelengthsLocation, wavelengths);

    const frequenciesLocation = gl.getUniformLocation(program, 'u_frequencies');
    gl.uniform1fv(frequenciesLocation, frequencies);

    gl.drawArrays(gl.TRIANGLE_STRIP, 0, 4);

    const ctx = overlayCanvas.getContext('2d');
    ctx.clearRect(0, 0, canvasWidth, canvasHeight);
    for (const source of sources) {
      ctx.fillStyle = source.color;
      ctx.beginPath();
      ctx.arc(source.x, source.y, sourceRadius, 0, 2 * Math.PI);
      ctx.fill();
      ctx.strokeStyle = 'white';
      ctx.lineWidth = 1;
      ctx.stroke();
    }
    if (draggingSourceId !== null) {
      const draggedSource = sources.find(s => s.id === draggingSourceId);
      if (draggedSource) {
        ctx.strokeStyle = 'yellow';
        ctx.lineWidth = 2;
        ctx.beginPath();
        ctx.arc(draggedSource.x, draggedSource.y, sourceRadius + 2, 0, 2 * Math.PI);
        ctx.stroke();
      }
    }
  }

  function animationLoop(timestamp) {
    currentTime = (timestamp - startTime) / 1000;
    drawVisualization(currentTime);
    animationFrameId = requestAnimationFrame(animationLoop);
  }

  function getRelativeCoords(event) {
    const rect = overlayCanvas.getBoundingClientRect();
    let clientX, clientY;
    if (event.touches && event.touches.length > 0) {
      clientX = event.touches[0].clientX;
      clientY = event.touches[0].clientY;
    } else if (event.changedTouches && event.changedTouches.length > 0) {
      clientX = event.changedTouches[0].clientX;
      clientY = event.changedTouches[0].clientY;
    } else {
      clientX = event.clientX;
      clientY = event.clientY;
    }
    const scaleX = overlayCanvas.width / rect.width;
    const scaleY = overlayCanvas.height / rect.height;
    const x = (clientX - rect.left) * scaleX;
    const y = (clientY - rect.top) * scaleY;
    return { x, y };
  }

  function handleMouseDown(event) {
    const { x, y } = getRelativeCoords(event);
    for (let i = sources.length - 1; i >= 0; i--) {
      const source = sources[i];
      const dist = distance(x, y, source.x, source.y);
      if (dist <= dragSensitivity) {
        draggingSourceId = source.id;
        dragOffsetX = x - source.x;
        dragOffsetY = y - source.y;
        isDragging = true;
        break;
      }
    }
  }

  function handleMouseMove(event) {
    if (!isDragging || draggingSourceId === null) return;
    const { x, y } = getRelativeCoords(event);
    const sourceIndex = sources.findIndex(s => s.id === draggingSourceId);
    if (sourceIndex !== -1) {
      sources[sourceIndex].x = Math.max(0, Math.min(canvasWidth, x - dragOffsetX));
      sources[sourceIndex].y = Math.max(0, Math.min(canvasHeight, y - dragOffsetY));
      sources = sources;
    }
  }

  function handleMouseUp(event) {
    if (isDragging) {
      draggingSourceId = null;
      isDragging = false;
    }
  }

  function handleTouchStart(event) {
    if (event.touches.length === 1) {
      event.preventDefault();
      handleMouseDown(event);
    }
  }

  function handleTouchMove(event) {
    if (isDragging) {
      event.preventDefault();
      handleMouseMove(event);
    }
  }

  function handleTouchEnd(event) {
    if (isDragging) {
      handleMouseUp(event);
    }
  }
</script>

<div class="navbar">
  <div class="logo">Wave interference simulation</div>
  <div class="navbar-links">
    <a href="https://github.com/0xdarkcap/wave-simulation" target="_blank">GitHub</a>
    <a href="https://varuns.info" target="_blank">Website</a>
</div>
</div>
<div class="simulation-container">
  <div class="controls">
    <!-- Source Controls Section -->
    {#each sources as source}
      <div class="source-controls">
        <h3>
          <span class="source-color" style="background-color: {source.color};"></span>
          Source {source.id}
        </h3>
        <div class="subsection">
          <label>
            Wavelength ({source.wavelength.toFixed(0)} px):
            <input
              type="range"
              min="10"
              max="150"
              step="1"
              bind:value={source.wavelength}
            />
          </label>
        </div>
        
        <div class="subsection">
          <label>
            Frequency ({source.frequency.toFixed(2)} Hz):
            <input
              type="range"
              min="0.1"
              max="2"
              step="0.05"
              bind:value={source.frequency}
            />
          </label>
          
        </div>
      </div>
    {/each}
  
    <!-- Separator between Source and Global -->
    <div class="section-divider"></div>
  
    <!-- Global Controls Section -->
    <div class="global-section">
      <h3>Global Settings</h3>
      <label>
        Amplitude Decay ({decayFactor.toFixed(2)}):
        <input
          type="range"
          min="0"
          max="2"
          step="0.05"
          bind:value={decayFactor}
        />
        <small>(0=None, 0.5=1/√r, 1=1/r)</small>
      </label>
      <label>
        Dot Density Factor ({dotDensityFactor.toFixed(2)}):
        <input
          type="range"
          min="0.1"
          max="3"
          step="0.05"
          bind:value={dotDensityFactor}
        />
        <small>(Adjusts brightness/contrast)</small>
      </label>
    </div>
  </div>
  <div class="canvas-container" bind:clientWidth={canvasWidth} bind:clientHeight={canvasHeight}>
    <canvas
      bind:this={webglCanvas}
      width={canvasWidth}
      height={canvasHeight}
      class="simulation-canvas webgl-canvas"
      aria-label="Wave interference simulation visualization"
    ></canvas>
    <canvas
      bind:this={overlayCanvas}
      width={canvasWidth}
      height={canvasHeight}
      class="simulation-canvas overlay-canvas"
      aria-label="Source overlay"
    ></canvas>
  </div>
  
</div>

<style>
    
    @import url('https://fonts.googleapis.com/css2?family=Special+Gothic+Expanded+One&display=swap');


:global(body) {
  margin: 0;
  font-family: "Special Gothic Expanded One", sans-serif;
  font-weight: 400;
  font-style: normal;

}

.controls {
  background-color: black;
  color: #e0e0e0;
  padding: 20px;
  height: calc(100vh - 90px);
  display: flex;
  flex-direction: column;
  gap: 20px;
}


/* Source Controls */
.source-controls {
  margin-top: 50px;
  padding: 0;
}

.source-controls h3 {
  margin: 0 0 15px 0;
  font-size: 1.2rem;
  font-weight: normal;
  color: #d4d4d4;
  display: flex;
  align-items: center;
  gap: 8px;
}

.source-color {
  width: 14px;
  height: 14px;
  display: inline-block;
  border: 1px solid #666;
}

.subsection {
  padding: 5px 0;
}

.divider {
  height: 1px;
  background: linear-gradient(to right, transparent, #666, transparent);
  margin: 10px 0;
}

/* Separator between Source and Global Sections */
.section-divider {
  height: 1px;
  background: linear-gradient(to right, transparent, #888, transparent);
  margin: 20px 0;
}

/* Global Section */
.global-section {
  padding: 0;
}

.global-section h3 {
  margin: 0 0 15px 0;
  font-size: 1.2rem;
  font-weight: normal;
  color: #d4d4d4;
}

/* Labels and Sliders */
label {
  display: block;
  margin-bottom: 15px;
  font-size: 0.9rem;
  color: #b0b0b0;
}

input[type="range"] {
  width: 100%;
  -webkit-appearance: none;
  appearance: none;
  height: 6px;
  background: #333;
  outline: none;
  margin-top: 8px;
  transition: background 0.3s ease;
}

input[type="range"]::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 16px;
  height: 16px;
  background: #e0e0e0;
  cursor: pointer;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.4);
  transition: transform 0.2s ease;
}

input[type="range"]::-moz-range-thumb {
  width: 16px;
  height: 16px;
  background: #e0e0e0;
  cursor: pointer;
  box-shadow: 0 2px 6px rgba(0, 0, 0, 0.4);
  transition: transform 0.2s ease;
}

input[type="range"]:hover::-webkit-slider-thumb,
input[type="range"]:hover::-moz-range-thumb {
  transform: scale(1.2);
}

input[type="range"]:hover {
  background: #555;
}

small {
  display: block;
  font-size: 0.7rem;
  color: #888;
  margin-top: 5px;
}
  body {
    margin: 0;
    padding: 0;
    overflow: hidden;
  }

  .navbar {
  background-color: #000;
  height: 50px;
  padding: 10px 20px;
  display: flex;
  flex-direction: row;
  justify-content: space-between;
  align-items: center;
  width: 100%;
  position: fixed;
    top: 0;
    left: 0;
    z-index: 4;
    border-bottom: 1px solid #fff;
  box-sizing: border-box;
}

.logo {
  font-family: 'Special Gothic Expanded One', sans-serif;
  font-size: 1.5rem;
  color: #fff;
  font-weight: normal;
}

.navbar-links {
  display: flex;
  flex-direction: row;
  gap: 20px;
}

.navbar-links a {
  font-family: 'Special Gothic Expanded One', sans-serif;
  font-size: 1rem;
  color: #fff;
  text-decoration: none;
  transition: color 0.3s ease;
}

.navbar-links a:hover {
  color: #ccc;
}

  .simulation-container {
    display: flex;
    flex-direction: row;
    width: 100vw;
    height: calc(100vh - 50px);
    margin-top: 50px;
  }

  .canvas-container {
    flex: 1;
    position: relative;
  }

  .simulation-canvas {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    border: 1px solid #555;
    background-color: transparent;
    display: block;
    cursor: grab;
    touch-action: none;
  }

  .webgl-canvas {
    z-index: 1;
  }

  .overlay-canvas {
    z-index: 2;
  }

  .simulation-canvas:active {
    cursor: grabbing;
  }

  @media (max-width: 768px) {
  .simulation-container {
    flex-direction: column;
  }
  .controls {
    height: auto;
    max-height: 50vh;
    overflow-y: auto;
    padding: 10px;
  }
  .canvas-container {
    height: auto;
    flex: 1;
  }
}
</style>