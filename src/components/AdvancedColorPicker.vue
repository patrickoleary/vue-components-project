<template>
  <div class="advanced-color-picker" :class="[currentTheme + '-theme']">
    <div class="color-input-row">
      <input type="color" class="compact-color-picker" v-model="backgroundColorValue" title="Selected Base Color" />
      <input type="color" id="histogram-color-input" class="compact-color-picker" v-model="histogramColor" title="Histogram Color">
      <input type="text" class="number-input" v-model="minValue" title="Minimum Value" />
      <select class="gradient-display" v-model="selectedColormapGradient" :style="{ background: selectedColormapGradient, color: 'transparent' }" title="Select Colormap">
        <option v-for="cmap in colormaps" :key="cmap.name" :value="cmap.gradient">
          {{ cmap.name }}
        </option>
      </select>
      <input type="text" class="number-input" v-model="maxValue" title="Maximum Value" />
      <div class="action-icon" @click="resetMinMaxValues" title="Reset Min/Max Values">↻</div>
    </div>
    <div class="main-picker-area" :style="{ backgroundColor: backgroundColorValue }">
      <svg
        v-if="transferFunctionMode === 'linear'"
        ref="linearEditorSvg"
        :width="svgEditorWidth"
        :height="svgEditorHeight"
        class="linear-editor-svg" 
        :style="{ backgroundColor: backgroundColorValue }"
        @mouseleave="handleEditorMouseLeave"
        @click="handleSvgBackgroundClick"
      >
        <defs>
          <linearGradient id="editorGradient" :x1="`${normalizedGradientStartX * 100}%`" :x2="`${normalizedGradientEndX * 100}%`" y1="0%" y2="0%" spreadMethod="pad">
            <stop
              v-for="(stop, index) in currentGradientStops"
              :key="'stop-' + index"
              :offset="stop.offset"
              :stop-color="stop.color"
            />
          </linearGradient>
        </defs>
        <g v-if="showHistogram && normalizedHistogram.length > 0" class="histogram-bars">
          <rect v-for="(bar, index) in normalizedHistogram"
                :key="'hist-bar-' + index"
                :x="bar.x_pixel"
                :y="bar.y_pixel"
                :width="bar.width_pixel"
                :height="bar.height_pixel"
                :fill="histogramColor" />
        </g>
        <polygon :points="editorPolygonPointsString" fill="url(#editorGradient)" :fill-opacity="polygonFillOpacity" />
        <rect
          x="0"
          y="0"
          :width="svgEditorWidth"
          :height="svgEditorHeight"
          class="svg-bg-rect"
          @click.prevent="handleSvgBackgroundClick"
        />
        <!-- Lines connecting points -->
        <line
          v-for="(line, index) in editorLines"
          :key="'line-' + index"
          :x1="line.x1" 
          :y1="line.y1"
          :x2="line.x2"
          :y2="line.y2"
          class="editor-line"
        />
        <!-- Points -->
        <circle
          v-for="(point, index) in linearPoints"
          :key="'point-' + index"
          :cx="normalizedToPixel(point).x"
          :cy="normalizedToPixel(point).y"
          r="5"
          class="control-point"
          :class="{ 'selected': index === selectedPointIndex }"
          :fill="index === selectedPointIndex ? 'green' : 'var(--control-point-color)'"
          :style="{ cursor: isDragging && draggingPointIndex === index ? 'grabbing' : 'grab' }"
          @mousedown.prevent.stop="handlePointMouseDown($event, index)"
          @dblclick.prevent.stop="handlePointDblClick($event, index)"
        />
      </svg>
      <svg
        v-if="transferFunctionMode === 'gaussian'"
        ref="gaussianEditorSvg"
        :width="svgEditorWidth"
        :height="svgEditorHeight"
        class="gaussian-editor-svg"
        :style="{ backgroundColor: backgroundColorValue }"
        @click.prevent="handleGaussianEditorBackgroundClick"
        @mouseup.prevent="handleGaussianDragMouseUp" 
        @mousemove.prevent="handleGaussianCenterPointMouseMove"
      >
        <defs>
          <linearGradient id="editorGradient" :x1="`${normalizedGradientStartX * 100}%`" :x2="`${normalizedGradientEndX * 100}%`" y1="0%" y2="0%" spreadMethod="pad">
            <stop
              v-for="(stop, index) in currentGradientStops"
              :key="'gaussian-grad-stop-' + index"
              :offset="stop.offset"
              :stop-color="stop.color"
            />
          </linearGradient>
        </defs>
        <!-- Gaussian editor content will go here -->
        <rect
          x="0"
          y="0"
          :width="svgEditorWidth"
          :height="svgEditorHeight"
          fill="transparent" 
        />
        <!-- Histogram -->
        <g v-if="showHistogram && normalizedHistogram.length > 0" class="histogram-bars">
          <rect v-for="(bar, index) in normalizedHistogram"
                :key="'gaussian-hist-bar-' + index" 
                :x="bar.x_pixel"
                :y="bar.y_pixel"
                :width="bar.width_pixel"
                :height="bar.height_pixel"
                :fill="histogramColor" />
        </g>

        <!-- Combined Gaussian Envelope Polygon -->
        <polygon
          :points="gaussianEditorEnvelopePointsString"
          :fill="'url(#editorGradient)'"
          :fill-opacity="polygonFillOpacity"
        />

        <!-- Gaussian Curves -->
        <path
          v-for="item in renderedGaussians"
          :key="'path-' + item.id"
          :d="item.path"
          class="gaussian-path"
          :stroke="currentTheme === 'light' ? '#333' : '#CCC'"
          stroke-width="2"
          fill="none"
        />
        <!-- Gaussian Control Points -->
        <circle
          v-for="item in renderedGaussians"
          :key="'point-' + item.id"
          :cx="item.controlPoint.x"
          :cy="item.controlPoint.y"
          r="5"
          class="gaussian-control-point"
          :class="{ 'selected': item.id === selectedGaussianId }"
          @mousedown.prevent.stop="handleGaussianPointMouseDown($event, item.id)"
          @dblclick.prevent="handleGaussianPointDblClick($event, item.id)"
        />
        <!-- Gaussian Width Handles -->
        <rect
          v-for="handle in selectedGaussianWidthHandles"
          :key="handle.id"
          :x="handle.x"
          :y="handle.y" 
          :width="handle.width"
          :height="handle.height"
          :fill="handle.fill" 
          class="gaussian-width-handle"
          @mousedown.stop="handleWidthHandleMouseDown($event, handle.gaussianId, handle.side)"
        />
      </svg>
      <div v-else-if="transferFunctionMode !== 'linear' && transferFunctionMode !== 'gaussian'" class="picker-area-placeholder">
        <span>Graphic editor for '{{ transferFunctionMode }}' mode is not yet implemented.</span>
      </div>
    </div>
    <div class="transfer-function-row">
      <div class="tf-control-group">
        <select id="tf-mode-select" v-model="transferFunctionMode" class="tf-select" title="Transfer Function Mode">
          <option v-for="mode in transferFunctionModes" :key="mode.value" :value="mode.value">
            {{ mode.text }}
          </option>
        </select>
      </div>

      <div class="tf-control-group">
        <input type="range" id="tf-opacity-slider" min="0" max="100" v-model.number="transferFunctionOpacity" class="tf-slider" title="Opacity">
        <span class="opacity-display">{{ transferFunctionOpacity }}%</span>
      </div>

      <div class="tf-control-group histogram-toggle-group">
        <button class="icon-toggle-button" :class="{ 'active-toggle': showHistogram }" @click="showHistogram = !showHistogram" title="Toggle Histogram Display">📊</button>
      </div>

      <div class="tf-control-group histogram-log-scale-group">
        <button class="icon-toggle-button" :class="{ 'active-toggle': useLogScale }" @click="useLogScale = !useLogScale" title="Use Log Scale for Histogram">📈</button>
      </div>

      </div>
  </div>
</template>

<script>
import colormapOptions from '@/assets/colormaps.json';

const GAUSSIAN_POINT_HIT_RADIUS_SQUARED = 10 * 10; // Pixel radius of 10, squared

export default {
  name: 'AdvancedColorPicker',
  data() {
    const defaultMinValue = '573.113';
    const defaultMaxValue = '886.808';
    return {
      backgroundColorValue: '#D3D3D3', /* Default to light grey */
      minValue: defaultMinValue,
      maxValue: defaultMaxValue,
      initialMinValue: defaultMinValue,
      initialMaxValue: defaultMaxValue,
      colormaps: colormapOptions,
      selectedColormapGradient: colormapOptions.length > 0 ? colormapOptions[0].gradient : '', // Initialize with the first colormap's gradient
      transferFunctionMode: 'linear',
      transferFunctionModes: [
        { value: 'linear', text: 'Linear' },
        { value: 'gaussian', text: 'Gaussian' }
      ],
      transferFunctionOpacity: 100,
      showHistogram: true,
      histogramColor: '#A9A9A9', /* Default to medium grey */
      currentTheme: 'light', // 'light' or 'dark'
      // Linear Transfer Function Editor Data
      linearPoints: [{ x: 0, y: 0 }, { x: 1, y: 1 }], // Normalized coordinates (0-1 range)
      selectedPointIndex: null,
      draggingPointIndex: null, // Index of the point being dragged
      draggedPointObject: null, // Actual object of the point being dragged
      isDragging: false,
      mouseDownTimeout: null, // For distinguishing click/dblclick from drag
      pendingDragIndex: null, // For distinguishing click/dblclick from drag
      initialMousePosition: { x: 0, y: 0 }, // Store mouse position on mousedown for drag threshold
      dragThreshold: 5, // Pixels mouse must move before a drag starts
      pointInteractionRecentlyActive: false, // To prevent bg click immediately after point interaction
      svgEditorWidth: 338, // Pixel width of the SVG drawing area
      svgEditorHeight: 150, // Pixel height of the SVG drawing area
      histogramData: [],
      histogramBins: 256,
      useLogScale: false,
      // Gaussian Transfer Function Editor Data
      gaussians: [], // Array of { id, c, h_control, b, w }
      selectedGaussianId: null,
      draggingGaussianId: null, // ID of the gaussian being dragged
      gaussianDragStartX: 0, // For dragging calculation
      gaussianDragStartY: 0,
      initialGaussianDragC: 0,
      initialGaussianDragH: 0, // For dragging calculation
      ignoreNextBackgroundClickForGaussian: false,
      nextGaussianId: 0,
      // New properties for width handles:
      activeDragMode: null, // null, 'center', or 'width'
      draggingWidthHandleSide: null, // 'left' or 'right'
      widthHandleInteractionActive: false, // Flag to prioritize width handle clicks
      widthHandleSize: 12, // Pixel size of the square width handles
    };
  },
  watch: {
    minValue(newValue, oldValue) {
      if (newValue === "") {
        // Allow field to be temporarily empty
        return;
      }
      const newMinValNum = parseFloat(newValue);

      if (isNaN(newMinValNum)) {
        // Not empty, but not a valid number (e.g., "abc")
        this.$nextTick(() => { this.minValue = oldValue; });
        return;
      }
      // No further validation on min < max here
    },
    maxValue(newValue, oldValue) {
      if (newValue === "") {
        // Allow field to be temporarily empty
        return;
      }
      const newMaxValNum = parseFloat(newValue);

      if (isNaN(newMaxValNum)) {
        // Not empty, but not a valid number (e.g., "abc")
        this.$nextTick(() => { this.maxValue = oldValue; });
        return;
      }
      // No further validation on min < max here
    },
  },
  mounted() {
    this.generateSampleHistogram();
  },
  computed: {
    gaussianEditorEnvelopePointsString() {
      const rawPoints = this.getGaussianEnvelopeRawPoints();
      return rawPoints.map(p => `${p.x},${p.y}`).join(' ');
    },

    selectedGaussianWidthHandles() {
      if (this.selectedGaussianId === null || typeof this.selectedGaussianId === 'undefined') { 
        return []; 
      }

      const selectedGaussian = this.gaussians.find(g => g.id === this.selectedGaussianId);

      if (!selectedGaussian) {
        return [];
      }

      const handles = [];
      const handleSize = this.widthHandleSize; // e.g., 6px
      const halfHandle = handleSize / 2;

      // Gaussian properties (normalized)
      const c = selectedGaussian.c; // Center x (cx)
      const h_control = selectedGaussian.h_control; // Peak Y (cy)
      const w = selectedGaussian.w; // Width
      // const b = selectedGaussian.b; // Baseline y - no longer used for handle Y

      // Left handle: center x is c - w, y is slightly offset from baseline (halfway)
      const leftHandleCenterNorm = { x: c - w, y: selectedGaussian.b + 0.025 };
      const leftHandlePixel = this.normalizedToPixel(leftHandleCenterNorm);
      handles.push({
        id: `${selectedGaussian.id}-width-left`,
        side: 'left',
        x: leftHandlePixel.x - halfHandle, // SVG rect x is top-left
        y: leftHandlePixel.y - halfHandle, // SVG rect y is top-left
        width: handleSize,
        height: handleSize,
        fill: 'green',
        gaussianId: selectedGaussian.id
      });

      // Right handle: center x is c + w, y is slightly offset from baseline (halfway)
      const rightHandleCenterNorm = { x: c + w, y: selectedGaussian.b + 0.025 };
      const rightHandlePixel = this.normalizedToPixel(rightHandleCenterNorm);
      handles.push({
        id: `${selectedGaussian.id}-width-right`,
        side: 'right',
        x: rightHandlePixel.x - halfHandle,
        y: rightHandlePixel.y - halfHandle,
        width: handleSize,
        height: handleSize,
        fill: 'green',
        gaussianId: selectedGaussian.id
      });
      
      return handles;
    },
    renderedGaussians() {
      const items = [];
      const samples = 100; // Number of points to sample for the curve

      this.gaussians.forEach(g => {
        let pathData = "M";
        const baseSigma = (g.w / 2.35482); // Normalized base sigma
        if (baseSigma < 1e-6) { // Avoid division by zero or tiny sigma if width is effectively zero
            // Draw a vertical line at c from b to h_control
            const p_start = this.normalizedToPixel({ x: g.c, y: g.b });
            const p_end = this.normalizedToPixel({ x: g.c, y: g.h_control });
            pathData += `${p_start.x},${p_start.y} L ${p_end.x},${p_end.y}`;
        } else {
            for (let i = 0; i <= samples; i++) {
                const x_norm = i / samples;
                let effectiveSigma = baseSigma;
                if (g.s && g.s !== 0) { // Check if s exists and is non-zero
                  if (x_norm < g.c) {
                    effectiveSigma = baseSigma * (1 - g.s); // Positive s narrows left side
                  } else {
                    effectiveSigma = baseSigma * (1 + g.s); // Positive s widens right side
                  }
                  effectiveSigma = Math.max(1e-6, effectiveSigma); // Prevent zero or negative sigma
                }
                // Gaussian equation for y_norm (0 at top, 1 at bottom)
                const y_norm_at_x = g.b + (g.h_control - g.b) * Math.exp(-Math.pow(x_norm - g.c, 2) / (2 * Math.pow(effectiveSigma, 2)));
                
                const pixel = this.normalizedToPixel({ x: x_norm, y: y_norm_at_x });
                pathData += `${pixel.x},${pixel.y}`;
                if (i < samples) {
                pathData += " L ";
                }
            }
        }

        items.push({
          id: g.id,
          path: pathData,
          controlPoint: this.normalizedToPixel({ x: g.c, y: g.h_control }),
          // Pass through original gaussian data if needed later
          c: g.c, 
          h_control: g.h_control,
          b: g.b,
          w: g.w
        });
      });
      return items;
    },
    currentGradientStops() {
      if (!this.selectedColormapGradient) return [];
      // Example: "linear-gradient(to right, #ff0000, #00ff00 50%, #0000ff)"
      // Simpler parsing for now, assuming colors are just listed or hex/rgb.
      // More robust parsing might be needed for complex CSS gradients.
      const gradientString = this.selectedColormapGradient;
      const colorRegex = /(#[0-9a-fA-F]{3,8}|rgba?\([^)]+\))/g;
      const colors = [];
      let match;
      while ((match = colorRegex.exec(gradientString)) !== null) {
        colors.push(match[0]);
      }

      if (colors.length === 0) return [];
      if (colors.length === 1) return [{ offset: '0%', color: colors[0] }, { offset: '100%', color: colors[0] }];

      return colors.map((color, index) => {
        const offset = (index / (colors.length - 1)) * 100;
        return {
          offset: `${offset}%`,
          color: color,
        };
      });
    },
    _initialMinNumeric() {
      return parseFloat(this.initialMinValue);
    },
    _initialMaxNumeric() {
      return parseFloat(this.initialMaxValue);
    },
    normalizedGradientStartX() {
      let currentMinNum = parseFloat(this.minValue);
      const initialMin = this._initialMinNumeric;
      const initialMax = this._initialMaxNumeric;

      if (isNaN(currentMinNum)) {
        currentMinNum = initialMin; // Default to initialMin if current minValue is empty or invalid
      }

      const dataSpan = initialMax - initialMin;
      if (dataSpan <= 0) return 0.0;

      let normStart = (currentMinNum - initialMin) / dataSpan;
      return Math.max(0, Math.min(1, normStart));
    },
    normalizedGradientEndX() {
      let currentMaxNum = parseFloat(this.maxValue);
      const initialMin = this._initialMinNumeric;
      const initialMax = this._initialMaxNumeric;

      if (isNaN(currentMaxNum)) {
        currentMaxNum = initialMax; // Default to initialMax if current maxValue is empty or invalid
      }

      const dataSpan = initialMax - initialMin;
      if (dataSpan <= 0) return 1.0;

      let normEnd = (currentMaxNum - initialMin) / dataSpan;
      return Math.max(0, Math.min(1, normEnd));
    },
    polygonFillOpacity() {
      return this.transferFunctionOpacity / 100;
    },
    editorPolygonPointsString() {
      const rawPoints = this.getLinearEnvelopeRawPoints();
      if (!rawPoints || rawPoints.length < 2) return ""; // Need at least 2 points for a line/polygon
      return rawPoints.map(p => `${p.x},${p.y}`).join(' ');
    },
    normalizedHistogram() {
      if (!this.histogramData || this.histogramData.length === 0 || this.histogramBins <= 0) {
        return [];
      }

      const barPixelWidth = this.svgEditorWidth / this.histogramBins;

      if (this.useLogScale) {
        const logTransformedData = this.histogramData.map(count => Math.log1p(count));
        const maxLogValue = Math.max(...logTransformedData);

        if (maxLogValue === 0) { // Handles if all original counts were 0
          return [];
        }

        const result = this.histogramData.map((count, index) => {
          const logValue = Math.log1p(count);
          const normalizedHeight = logValue / maxLogValue;
          const pixelHeight = normalizedHeight * this.svgEditorHeight;
          const x_pixel = index * barPixelWidth;
          const y_pixel = this.svgEditorHeight - pixelHeight;
          return {
            x_pixel,
            y_pixel,
            width_pixel: barPixelWidth,
            height_pixel: pixelHeight,
          };
        });
        return result;

      } else {
        // Linear Scale
        const maxEntryValue = Math.max(...this.histogramData);

        if (maxEntryValue === 0) { // Handles if all counts are 0
          return [];
        }

        const result = this.histogramData.map((count, index) => {
          const normalizedHeight = count / maxEntryValue; // Proportion of max entry
          const pixelHeight = normalizedHeight * this.svgEditorHeight;
          const x_pixel = index * barPixelWidth;
          const y_pixel = this.svgEditorHeight - pixelHeight;
          return {
            x_pixel,
            y_pixel,
            width_pixel: barPixelWidth,
            height_pixel: pixelHeight,
          };
        });
        return result;
      }
      // This log is now inside the if/else blocks
      // console.log('normalizedHistogram computed:', JSON.parse(JSON.stringify(result)));
      // return result;
    },
    editorLines() {
      if (this.linearPoints.length < 2) {
        return [];
      }
      // linearPoints are normalized, convert to pixel for line rendering
      const lines = [];
      for (let i = 0; i < this.linearPoints.length - 1; i++) {
        const p1 = this.normalizedToPixel(this.linearPoints[i]);
        const p2 = this.normalizedToPixel(this.linearPoints[i+1]);
        lines.push({
          x1: p1.x,
          y1: p1.y,
          x2: p2.x,
          y2: p2.y,
        });
      }
      return lines;
    }
  },
  beforeUnmount() {
    // Clean up global event listeners to prevent memory leaks or errors
    window.removeEventListener('mousemove', this.handleEditorMouseMove);
    window.removeEventListener('mouseup', this.handleEditorMouseUp);
    window.removeEventListener('mousemove', this.handleGaussianCenterPointMouseMove); // From Gaussian editor
    window.removeEventListener('mouseup', this.handleGaussianDragMouseUp); // From Gaussian editor
    window.removeEventListener('mousemove', this.handleGaussianWidthHandleMouseMove); // From Gaussian editor
  },
  methods: {
    getGaussianEnvelopeRawPoints() {
      const bottomYPixel = this.normalizedToPixel({ x: 0, y: 0 }).y; // y=0 normalized is bottom of graph
      const startPixelPoint = { x: this.normalizedToPixel({ x: 0, y: 0 }).x, y: bottomYPixel };
      const endPixelPoint = { x: this.normalizedToPixel({ x: 1, y: 0 }).x, y: bottomYPixel };

      if (!this.gaussians || this.gaussians.length === 0) {
        return [startPixelPoint, endPixelPoint];
      }

      const envelopePixelPoints = [];
      envelopePixelPoints.push(startPixelPoint);

      const numSamples = Math.max(2, Math.floor(this.svgEditorWidth / 2));

      for (let i = 0; i <= numSamples; i++) {
        const x_norm = i / numSamples;
        let combined_y_norm_at_x = 0.0; 

        this.gaussians.forEach(g => {
          const baseSigma = (g.w / 2.35482);
          let y_gaussian_at_x;
          if (baseSigma < 1e-6) { // Effectively zero width, treat as a spike
            y_gaussian_at_x = (Math.abs(x_norm - g.c) < (0.5 / this.svgEditorWidth)) ? g.h_control : g.b;
          } else {
            let effectiveSigma = baseSigma;
            if (g.s && g.s !== 0) { // Check if s exists and is non-zero
              if (x_norm < g.c) {
                effectiveSigma = baseSigma * (1 - g.s); // Positive s narrows left side
              } else {
                effectiveSigma = baseSigma * (1 + g.s); // Positive s widens right side
              }
              effectiveSigma = Math.max(1e-6, effectiveSigma); // Prevent zero or negative sigma
            }
            const amplitudeFactor = g.h_control - g.b; // h_control is peak, b is baseline
            const exponent = -Math.pow(x_norm - g.c, 2) / (2 * effectiveSigma * effectiveSigma);
            y_gaussian_at_x = g.b + amplitudeFactor * Math.exp(exponent);
          }
          combined_y_norm_at_x = Math.max(combined_y_norm_at_x, y_gaussian_at_x);
        });
        
        combined_y_norm_at_x = Math.max(0, Math.min(1, combined_y_norm_at_x));
        const pixelP = this.normalizedToPixel({ x: x_norm, y: combined_y_norm_at_x });
        envelopePixelPoints.push({ x: pixelP.x, y: pixelP.y });
      }

      envelopePixelPoints.push(endPixelPoint);
      return envelopePixelPoints;
    },
    getLinearEnvelopeRawPoints() {
      const bottomLeftPixel = this.normalizedToPixel({ x: 0, y: 0 });
      const bottomRightPixel = this.normalizedToPixel({ x: 1, y: 0 });

      // Ensure linearPoints are sorted by x for correct polygon shape
      const sortedNormalizedPoints = [...this.linearPoints].sort((a, b) => a.x - b.x);
      
      const pixelControlPoints = sortedNormalizedPoints.map(p => {
        return this.normalizedToPixel(p); // p already has x, y from this.linearPoints
      });
      
      // The polygon should be: bottom-left, then all user points in order, then bottom-right.
      // SVG will auto-close from bottom-right to bottom-left.
      return [bottomLeftPixel, ...pixelControlPoints, bottomRightPixel];
    },

    toggleTheme() {
      this.currentTheme = this.currentTheme === 'light' ? 'dark' : 'light';
    },
    resetMinMaxValues() {
      this.minValue = this.initialMinValue;
      this.maxValue = this.initialMaxValue;
    },

    // Basic contrast checker to make text visible on gradient. 
    // This is a simplified version and might not be perfect for all gradients.
    kontrastColor(gradientString) {
      // Try to get a color from the middle of the gradient
      // This is a heuristic and might not be robust
      try {
        const colors = gradientString.match(/rgb\([^)]+\)|#[0-9a-fA-F]{3,6}/g);
        if (colors && colors.length > 0) {
          // Use the middle color, or the first if only one/two
          const sampleColor = colors[Math.floor(colors.length / 2)]; 
          let r, g, b;
          if (sampleColor.startsWith('#')) {
            const hex = sampleColor.replace('#', '');
            if (hex.length === 3) {
              r = parseInt(hex[0] + hex[0], 16);
              g = parseInt(hex[1] + hex[1], 16);
              b = parseInt(hex[2] + hex[2], 16);
            } else {
              r = parseInt(hex.substring(0, 2), 16);
              g = parseInt(hex.substring(2, 4), 16);
              b = parseInt(hex.substring(4, 6), 16);
            }
          } else { // rgb()
            const parts = sampleColor.match(/\d+/g);
            r = parseInt(parts[0]);
            g = parseInt(parts[1]);
            b = parseInt(parts[2]);
          }
          // Simple brightness check
          const brightness = (r * 299 + g * 587 + b * 114) / 1000;
          return brightness > 128 ? '#000000' : '#FFFFFF';
        }
      } catch (e) { /* ignore errors, fallback */ }
      return '#FFFFFF'; // Default to white text
    },

    // --- Linear Transfer Function Editor Methods ---
    normalizedToPixel(normalizedPoint) {
      if (!normalizedPoint) return { x: 0, y: 0 };
      return {
        x: normalizedPoint.x * this.svgEditorWidth,
        y: (1 - normalizedPoint.y) * this.svgEditorHeight, // Invert Y for SVG coordinates
      };
    },

    pixelToNormalized(pixelPoint) {
      if (!pixelPoint) return { x: 0, y: 0 };
      const normX = this.svgEditorWidth === 0 ? 0 : pixelPoint.x / this.svgEditorWidth;
      // Invert Y from SVG coordinates back to normalized graph coordinates
      const normY = this.svgEditorHeight === 0 ? 0 : 1 - (pixelPoint.y / this.svgEditorHeight);
      return {
        x: Math.max(0, Math.min(1, normX)),
        y: Math.max(0, Math.min(1, normY)), // Ensure Y stays within 0-1 after inversion
      };
    },

    getSvgCoordinates(event, svgElement) {
      if (!svgElement) {
        return null;
      }
      // Prioritize offsetX/offsetY if the event target is the SVG itself.
      if (event.target === svgElement && event.offsetX !== undefined && event.offsetY !== undefined) {
        return {
          x: event.offsetX,
          y: event.offsetY
        };
      }
      // Fallback to getBoundingClientRect for other cases (e.g., if event target is a child of SVG)
      // or if offsetX/Y are not available on the event object for some reason.
      const svgRect = svgElement.getBoundingClientRect();
      if (typeof event.clientX !== 'undefined' && typeof event.clientY !== 'undefined') {
        return {
          x: event.clientX - svgRect.left,
          y: event.clientY - svgRect.top
        };
      }
      // Final, less reliable fallback if clientX/Y are also missing.
      return {
          x: (event.offsetX !== undefined) ? event.offsetX : (event.layerX - svgElement.clientLeft),
          y: (event.offsetY !== undefined) ? event.offsetY : (event.layerY - svgElement.clientTop)
      };
    },

    addPoint(x, y) {
      const newPoint = { x: Math.max(0, Math.min(1, x)), y: Math.max(0, Math.min(1, y)) };
      this.linearPoints.push(newPoint);
      this.linearPoints.sort((a, b) => a.x - b.x);
      this.selectedPointIndex = this.linearPoints.indexOf(newPoint);
    },

    handleSvgBackgroundClick(event) {
      // Check 1: Click originated on an existing control point. If so, do nothing here.
      if (event.target.closest && event.target.closest('.control-point')) {
        return;
      }

      // Check 2: If a drag just finished, or a point interaction was very recent (e.g., dblclick),
      // or currently dragging, do nothing further on this background click.
      if (this.pointInteractionRecentlyActive || this.isDragging) {
        return;
      }

      // If we're here, it's a "clean" click on the SVG background.
      if (this.selectedPointIndex !== null) {
        // A point is currently selected, so this background click should deselect it.
        this.selectedPointIndex = null;
      } else {
        // No point is selected, so create a new one.
        const pixelCoords = this.getSvgCoordinates(event, this.$refs.linearEditorSvg);
        if (!pixelCoords) return; 
        const normalizedCoords = this.pixelToNormalized(pixelCoords);
        this.addPoint(normalizedCoords.x, normalizedCoords.y);
      }
    },

    handlePointMouseDown(event, index) {
      this.pointInteractionRecentlyActive = true;
      this.selectedPointIndex = index;
      this.initialMousePosition = this.getSvgCoordinates(event, this.$refs.linearEditorSvg);
      
      this.isDragging = false; // Not dragging yet
      this.draggingPointIndex = null;
      this.draggedPointObject = null;

      // Add global listeners
      window.addEventListener('mousemove', this.handleEditorMouseMove);
      window.addEventListener('mouseup', this.handleEditorMouseUp);
    },

    handleEditorMouseMove(event) {
      // Use this.$refs.linearEditorSvg explicitly for getSvgCoordinates context
      // Ensure a point interaction was initiated.
      if (this.selectedPointIndex === null && !this.isDragging) {
        // If no point was mousedowned, or if a drag was already cancelled but listeners somehow persisted.
        window.removeEventListener('mousemove', this.handleEditorMouseMove);
        window.removeEventListener('mouseup', this.handleEditorMouseUp);
        return;
      }

      const pixelCoords = this.getSvgCoordinates(event, this.$refs.linearEditorSvg);
      if (!pixelCoords) return; // Mouse might be outside a trackable area for SVG conversion

      if (!this.isDragging) {
        // Not dragging yet, check threshold if a point is selected.
        if (this.selectedPointIndex !== null && this.initialMousePosition) {
          const dx = pixelCoords.x - this.initialMousePosition.x;
          const dy = pixelCoords.y - this.initialMousePosition.y;
          if (Math.sqrt(dx * dx + dy * dy) > this.dragThreshold) {
            this.isDragging = true;
            this.draggingPointIndex = this.selectedPointIndex;
            // Ensure the point still exists in the array
            if (this.draggingPointIndex >= 0 && this.draggingPointIndex < this.linearPoints.length) {
                this.draggedPointObject = this.linearPoints[this.draggingPointIndex];
            } else {
                // Point disappeared or index out of bounds - critical state error, clean up.
                this.isDragging = false;
                this.selectedPointIndex = null;
                this.draggingPointIndex = null;
                this.draggedPointObject = null;
                window.removeEventListener('mousemove', this.handleEditorMouseMove);
                window.removeEventListener('mouseup', this.handleEditorMouseUp);
                return;
            }
          } else {
            return; // Not enough movement to start drag.
          }
        } else {
          // No selected point for drag initiation, or initial position missing. This shouldn't normally be hit if selectedPointIndex is null check above is working.
          return;
        }
      }

      // If dragging, update point position.
      if (this.isDragging && this.draggedPointObject) {
        event.preventDefault(); // Prevent text selection, etc., during drag
        const normalizedCoords = this.pixelToNormalized(pixelCoords);
        const point = this.draggedPointObject;
        const currentIndex = this.linearPoints.indexOf(point);

        if (currentIndex === -1) { // Point somehow removed during drag
            this.handleEditorMouseUp(event); // Treat as drag end and clean up
            return;
        }

        const isFirstPoint = currentIndex === 0;
        const isLastPoint = currentIndex === this.linearPoints.length - 1;

        if (isFirstPoint) {
          point.x = 0;
          point.y = Math.max(0, Math.min(1, normalizedCoords.y));
        } else if (isLastPoint) {
          point.x = 1;
          point.y = Math.max(0, Math.min(1, normalizedCoords.y));
        } else {
          const prevPointX = this.linearPoints[currentIndex - 1].x;
          const nextPointX = this.linearPoints[currentIndex + 1].x;
          const epsilon = 0.0001;
          point.x = Math.max(prevPointX + epsilon, Math.min(normalizedCoords.x, nextPointX - epsilon));
          point.y = Math.max(0, Math.min(1, normalizedCoords.y));
        }
        point.x = Math.max(0, Math.min(1, point.x)); // Redundant clamp for x, but safe.
      } else if (this.isDragging && !this.draggedPointObject) {
        // Dragging flag is true, but no object. Inconsistent state. Clean up.
        this.handleEditorMouseUp(event);
      }
    },

    handleEditorMouseUp(event) {
      // This is now a global mouseup handler.
      const wasDragging = this.isDragging; // Store drag state before reset
      // This is now a global mouseup handler.
      window.removeEventListener('mousemove', this.handleEditorMouseMove);
      window.removeEventListener('mouseup', this.handleEditorMouseUp);
      
      if (event) event.preventDefault(); // Prevent any default action if event is passed

      if (this.isDragging && this.draggedPointObject) {
        // Ensure point still exists before sorting and finding index
        if (this.linearPoints.includes(this.draggedPointObject)) {
            this.linearPoints.sort((a, b) => a.x - b.x);
            // Update selectedPointIndex to the final position of the dragged point
            this.selectedPointIndex = this.linearPoints.indexOf(this.draggedPointObject);
        } else {
            // Point was deleted during drag, clear selection
            this.selectedPointIndex = null;
        }
      }
      // If not dragging (isDragging was false), selectedPointIndex remains as set by mousedown (simple click).
      // If selectedPointIndex is null (e.g. mousedown on background then mouseup), it remains null.

      this.isDragging = false;
      this.draggingPointIndex = null;
      this.draggedPointObject = null;
      // initialMousePosition is not reset here; it's set by the next mousedown.
      // selectedPointIndex is intentionally NOT reset here if it was just a click.

      // If it was a click (not a drag that moved the point) on a point, selectedPointIndex is already set.
      // If it was a drag, selectedPointIndex was updated to the new position.
      // If it was a background click, selectedPointIndex is null.

      this.$nextTick(() => {
        this.pointInteractionRecentlyActive = false;
      });
    },
    
    handleEditorMouseLeave(event) {
      // If a point interaction was started (mousedown on a point, so selectedPointIndex is set)
      // or if a drag is in progress, treat leaving the SVG as a mouseup event to finalize/cancel.
      if (this.selectedPointIndex !== null || this.isDragging) {
          this.handleEditorMouseUp(event); 
      }
      // If no interaction was active (e.g. mouse just passing over), do nothing.
    },

    handlePointDblClick(event, index) {
      this.pointInteractionRecentlyActive = true; // Treat dblclick as an interaction
      // Ensure any potential drag operation is fully cancelled
      this.isDragging = false;
      this.draggingPointIndex = null;
      this.draggedPointObject = null;
      // Clear any old timeout logic if it's still somehow present
      if (this.mouseDownTimeout) {
        clearTimeout(this.mouseDownTimeout);
        this.mouseDownTimeout = null;
        this.pendingDragIndex = null;
      }

      if (index < 0 || index >= this.linearPoints.length) return;
      if (index === 0 || index === this.linearPoints.length - 1) {
        return;
      }

      // If the point being double-clicked was the selected one, deselect it.
      if (this.selectedPointIndex === index) {
        this.selectedPointIndex = null;
      }
      
      this.linearPoints.splice(index, 1);

      // Adjust selectedPointIndex if a preceding point was removed and a different point is selected
      if (this.selectedPointIndex !== null && this.selectedPointIndex > index) {
        this.selectedPointIndex--;
      }
      // Ensure interaction flag is reset after dblclick processing
      this.$nextTick(() => {
        this.pointInteractionRecentlyActive = false;
      });
    },
    setHistogramData(data, bins = 256) {
      this.histogramData = data;
      this.histogramBins = bins;
    },
    generateSampleHistogram() {
      // Use this.histogramBins which is initialized to 256
      const sampleData = new Array(this.histogramBins).fill(0).map(() => Math.floor(Math.random() * 100)); // Example integer counts
      this.setHistogramData(sampleData, this.histogramBins);
    },
    // --- End Linear Transfer Function Editor Methods ---

    // --- Gaussian Transfer Function Editor Methods ---
    handleGaussianEditorBackgroundClick(event) {
      if (this.ignoreNextBackgroundClickForGaussian) {
        this.ignoreNextBackgroundClickForGaussian = false; // Consume the flag for this click
        return;
      }
      if (!this.$refs.gaussianEditorSvg) return;

      const clickCoords = this.getSvgCoordinates(event, this.$refs.gaussianEditorSvg);
      if (!clickCoords) return;

      // Check if the click is near an existing control point
      for (const g of this.gaussians) {
        const pointPixel = this.normalizedToPixel({ x: g.c, y: g.h_control });
        const dx = clickCoords.x - pointPixel.x;
        const dy = clickCoords.y - pointPixel.y;
        const distSq = dx * dx + dy * dy;

        if (distSq <= GAUSSIAN_POINT_HIT_RADIUS_SQUARED) {
          // Click is near an existing point. Select it and do not add a new one.
          this.selectedGaussianId = g.id;
          // If the original event target was the SVG, but we're treating this as a point interaction,
          // we might want to stop propagation to prevent other SVG-level handlers if any existed.
          event.stopPropagation(); 
          return;
        }
      }

      // If not near any point, proceed to add a new Gaussian (original logic continues from here)



      const normalizedCoords = this.pixelToNormalized(clickCoords);

      const newGaussian = {
        id: this.nextGaussianId++,
        c: normalizedCoords.x,       // Center (0-1, normalized)
        h_control: normalizedCoords.y, // Peak Y (0-1, normalized, 0 is top)
        b: 0.0,                      // Baseline Y (0-1, normalized, 0 is top)
        w: 0.1,                      // Width FWHM (0-1, normalized, proportion of editor width)
        s: 0.0,                      // Skewness (-1 to 1, 0 for no skew, positive skews right)
      };
      this.gaussians.push(newGaussian);
      this.selectedGaussianId = newGaussian.id; // Select the new Gaussian
      // console.log('Added Gaussian:', newGaussian, this.gaussians);
    },
    handleGaussianPointMouseDown(event, gaussianId) {
      // console.log(`[GP MD] ID: ${gaussianId}, widthHandleActive: ${this.widthHandleInteractionActive}`); // Debug log removed
      if (this.widthHandleInteractionActive) return; // Prioritize width handle interaction if active
      event.stopPropagation();
      this.selectedGaussianId = gaussianId;
      this.draggingGaussianId = gaussianId;
      this.activeDragMode = 'center'; // Indicate center point dragging
      const currentGaussian = this.gaussians.find(g => g.id === gaussianId);
      if (currentGaussian && this.$refs.gaussianEditorSvg) {
        const mousePos = this.getSvgCoordinates(event, this.$refs.gaussianEditorSvg);
        this.gaussianDragStartX = mousePos.x;
        this.gaussianDragStartY = mousePos.y;
        this.initialGaussianDragC = currentGaussian.c;
        this.initialGaussianDragH = currentGaussian.h_control;

        // Add mouse move and up listeners to the window to capture drags outside SVG
        window.addEventListener('mousemove', this.handleGaussianCenterPointMouseMove);
        window.addEventListener('mouseup', this.handleGaussianDragMouseUp);
      }
    },

    handleGaussianCenterPointMouseMove(event) {
      // console.log(`[GCP MM] Entry - Dragging ID: ${this.draggingGaussianId}, Mode: ${this.activeDragMode}`); // Debug log removed
      if (this.draggingGaussianId === null || !this.$refs.gaussianEditorSvg || this.activeDragMode !== 'center') {
        // console.log('[GCP MM] Bailing: no drag ID, no SVG, or wrong mode.');
        return;
      }
      // console.log(`[GCP MM] Dragging ID: ${this.draggingGaussianId}, Mode: ${this.activeDragMode} - Proceeding`); // Redundant with entry log
      event.preventDefault();

      const gaussian = this.gaussians.find(g => g.id === this.draggingGaussianId);
      if (!gaussian) return;

      const mousePos = this.getSvgCoordinates(event, this.$refs.gaussianEditorSvg);
      if (!mousePos) {
        // If mouse is outside SVG, we might want to stop dragging or clamp
        // For now, let's use last known good coordinates if possible or stop
        return; 
      }

      const deltaX = mousePos.x - this.gaussianDragStartX;
      const deltaY = mousePos.y - this.gaussianDragStartY;

      // Convert pixel deltas to normalized deltas
      // A change of svgEditorWidth pixels in X is a change of 1.0 in normalized C
      // A change of svgEditorHeight pixels in Y is a change of 1.0 in normalized H
      const normalizedDeltaC = deltaX / this.svgEditorWidth;
      const normalizedDeltaH = deltaY / this.svgEditorHeight;

      let newC = this.initialGaussianDragC + normalizedDeltaC;
      let newH = this.initialGaussianDragH - normalizedDeltaH;

      // Clamp c and h_control to be within [0, 1]
      newC = Math.max(0, Math.min(1, newC));
      newH = Math.max(0, Math.min(1, newH));

      gaussian.c = newC;
      gaussian.h_control = newH;
      // console.log(`[GCP MM] Updated Gaussian ${gaussian.id}: c=${newC}, h_control=${newH}`); // Log is for entry, not update here
    },

    handleGaussianDragMouseUp(event) {
      // console.log(`[G MU] Entry - Dragging ID: ${this.draggingGaussianId}, Mode: ${this.activeDragMode} (before reset)`); // Debug log removed
      if (this.draggingGaussianId !== null) {
        // Ensure the Gaussian remains selected
        this.selectedGaussianId = this.draggingGaussianId;

        // Set flag to prevent immediate new Gaussian creation on background click.
        // This flag will be consumed by the next call to handleGaussianEditorBackgroundClick.
        this.ignoreNextBackgroundClickForGaussian = true;
      }

      // Reset all common and mode-specific dragging states
      this.draggingGaussianId = null;
      this.activeDragMode = null;
      this.draggingWidthHandleSide = null;
      // Optional: Reset initial drag values if desired, though they are set on mousedown
      // this.gaussianDragStartX = 0;
      // this.gaussianDragStartY = 0;
      // this.initialGaussianDragC = 0;
      // this.initialGaussianDragH = 0;
      // this.initialGaussianDragW = 0;
      // this.widthHandleDragStartX = 0;

      // Remove all potentially active global listeners
      window.removeEventListener('mousemove', this.handleGaussianCenterPointMouseMove);
      window.removeEventListener('mousemove', this.handleGaussianWidthHandleMouseMove);
      window.removeEventListener('mouseup', this.handleGaussianDragMouseUp);
    },

    handleWidthHandleMouseDown(event, gaussianId, side) {
      // console.log(`[WH MD] ID: ${gaussianId}, Side: ${side}, widthHandleActive (before): ${this.widthHandleInteractionActive}`); // Debug log removed
      this.widthHandleInteractionActive = true;
      event.stopPropagation();
      const currentGaussian = this.gaussians.find(g => g.id === gaussianId);
      if (!currentGaussian || !this.$refs.gaussianEditorSvg) return;

      this.selectedGaussianId = gaussianId;
      this.draggingGaussianId = gaussianId;
      this.activeDragMode = 'width';
      this.draggingWidthHandleSide = side; // Retained for clarity, though not used by current width calc.
      
      // Add global listeners for width dragging
      window.addEventListener('mousemove', this.handleGaussianWidthHandleMouseMove);
      window.addEventListener('mouseup', this.handleGaussianDragMouseUp);

      this.$nextTick(() => {
        this.widthHandleInteractionActive = false;
      });
    },

    handleGaussianWidthHandleMouseMove(event) {
      if (this.draggingGaussianId === null || this.activeDragMode !== 'width' || !this.$refs.gaussianEditorSvg) {
        return;
      }
      event.preventDefault();

      const currentGaussian = this.gaussians.find(g => g.id === this.draggingGaussianId);
      if (!currentGaussian) {
        return;
      }

      const mousePos = this.getSvgCoordinates(event, this.$refs.gaussianEditorSvg);
      if (!mousePos) {
        return;
      }

      const normalizedMouseX = this.pixelToNormalized({ x: mousePos.x, y: 0 }).x;
      const initialW = currentGaussian.w; // Not strictly needed anymore but kept for clarity if debugging returns
      let calculatedNewW = Math.abs(normalizedMouseX - currentGaussian.c);

      const minNormalizedW = 0.01; // Minimum width (FWHM / 2)
      const maxNormalizedW = 0.5;  // Maximum width (FWHM / 2)
      const finalW = Math.max(minNormalizedW, Math.min(maxNormalizedW, calculatedNewW));
      
      currentGaussian.w = finalW;
    },

    handleGaussianPointDblClick(event, gaussianId) {
      event.stopPropagation();
      this.gaussians = this.gaussians.filter(g => g.id !== gaussianId);
      if (this.selectedGaussianId === gaussianId) {
        this.selectedGaussianId = null;
      }
      if (this.draggingGaussianId === gaussianId) {
        this.draggingGaussianId = null;
        this.activeDragMode = null;
        this.draggingWidthHandleSide = null;
        // Remove global listeners in case a drag was interrupted by dblclick
        window.removeEventListener('mousemove', this.handleGaussianCenterPointMouseMove);
        window.removeEventListener('mousemove', this.handleGaussianWidthHandleMouseMove);
        window.removeEventListener('mouseup', this.handleGaussianDragMouseUp);
      }
    },
    // --- End Gaussian Transfer Function Editor Methods ---


    getNormalizedLinearPoints() {
      // Return a deep copy to prevent external modification of internal state
      return JSON.parse(JSON.stringify(this.linearPoints));
    }
  }
};
</script>

<style scoped>
/* THEME VARIABLE DEFINITIONS */
.advanced-color-picker.light-theme {
  --bg-color: #F0F0F0;
  --text-color: #333333;
  --border-color: #CCCCCC;
  --section-bg-color: #FFFFFF;
  --input-bg-color: #FFFFFF;
  --input-text-color: #333333;
  --input-border-color: #BBBBBB;
  --select-option-bg-color: #EAEAEA;
  --select-option-text-color: #333333;
  --action-icon-color: #555555;
  --action-icon-hover-color: #000000;
  --main-picker-area-placeholder-text-color: #a0a0a0;
  --control-point-color: var(--action-icon-color); /* Default for linear editor points */
}

.advanced-color-picker.dark-theme {
  --bg-color: #333333;
  --text-color: #CCCCCC;
  --border-color: #444444;
  --section-bg-color: #4A4A4A;
  --input-bg-color: #2E2E2E;
  --input-text-color: #E0E0E0;
  --input-border-color: #222222;
  --select-option-bg-color: #3E3E3E;
  --select-option-text-color: #E0E0E0;
  --action-icon-color: #AAAAAA;
  --action-icon-hover-color: #FFFFFF;
  --main-picker-area-placeholder-text-color: #888888;
}
/* END THEME VARIABLE DEFINITIONS */

.advanced-color-picker {
  width: 350px; /* Approximate width, can be adjusted */
  box-sizing: border-box; /* Ensure padding and border are included in the width */
  background-color: var(--bg-color);
  color: var(--text-color);
  border: 1px solid var(--border-color);
  border-radius: 4px;
  padding: 5px;
  font-family: sans-serif;
  display: flex;
  flex-direction: column;
  gap: 5px; /* Space between sections */
}

.color-input-row,
.main-picker-area,
.transfer-function-row,
.bottom-icon-bar {
  background-color: var(--section-bg-color);
}

.transfer-function-row {
  display: flex;
  justify-content: space-between; /* Distribute items evenly */
  flex-wrap: nowrap; /* Explicitly prevent wrapping */
  gap: 6px; /* Further reduced gap */
  padding: 10px 8px; /* Adjusted padding */
  align-items: center; /* Ensure vertical alignment */
}

.tf-control-group {
  display: flex;
  align-items: center;
  gap: 5px; /* Space between label and control within a group */
}

.tf-select,
.tf-slider,
.tf-checkbox {
  /* .tf-color-input styles moved to .compact-color-picker */
  background-color: var(--input-bg-color);
  color: var(--input-text-color);
  border: 1px solid var(--input-border-color);
  border-radius: 3px;
  padding: 3px 5px;
  font-size: 0.85em;
  box-sizing: border-box;
}

.tf-select {
  min-width: 70px; /* Further reduced min-width */
  height: 24px;
}

.tf-slider {
  flex-grow: 1; /* Allow slider to take available space */
  min-width: 60px; /* Ensure it doesn't get too small */
  /* width: 75px; */ /* Removed fixed width */
  height: 18px; /* Standard height for range inputs */
  padding: 0; /* Remove padding for range */
}

.opacity-display {
  font-size: 0.8em;
  min-width: 30px; /* Ensure space for '100%' */
  text-align: right;
}

.icon-toggle-button {
  background-color: transparent;
  border: 1px solid var(--input-border-color);
  color: var(--action-icon-color);
  padding: 3px;
  border-radius: 3px;
  cursor: pointer;
  font-size: 1em; /* Adjust if icons are too small/large */
  width: 26px;
  height: 26px;
  display: flex;
  align-items: center;
  justify-content: center;
  box-sizing: border-box;
}

.icon-toggle-button:hover {
  background-color: var(--input-bg-color);
  border-color: var(--action-icon-hover-color);
  color: var(--action-icon-hover-color);
}

.icon-toggle-button.active-toggle {
  background-color: var(--action-icon-color);
  color: var(--section-bg-color);
  border-color: var(--action-icon-color);
}

.icon-toggle-button.active-toggle:hover {
  background-color: var(--action-icon-hover-color);
  color: var(--section-bg-color);
  border-color: var(--action-icon-hover-color);
}

.compact-color-picker {
  width: 24px;
  height: 24px;
  padding: 1px; /* Minimal padding for color input - adjust if browser adds too much internal padding */
  border: 1px solid var(--input-border-color);
  background-color: transparent; /* Allows the browser's color picker UI to show through */
  cursor: pointer;
  border-radius: 3px;
  box-sizing: border-box; /* Ensures padding and border are included in width/height */
  /* display: flex and text-align properties are generally not needed for <input type="color"> */
}

.main-picker-area {
  min-height: 150px; /* Larger area for the main picker */
}

.main-picker-area span {
  display: flex;
  justify-content: center;
  align-items: center;
  height: 100%;
  color: var(--main-picker-area-placeholder-text-color);
  user-select: none;
}

.color-input-row {
  display: flex;
  justify-content: space-between;
  align-items: center; /* Ensure vertical alignment */
  padding: 6px 8px; /* Adjusted padding for content */
  flex-wrap: nowrap; /* Ensure it stays on one line */
  gap: 4px; /* Add a small gap between items */
  /* min-height is already set */
}

.color-swatch {
  width: 22px;
  height: 22px;
  background-color: #777; /* Placeholder initial color, can be bound to data later */
  border: 1px solid #2c2c2c;
  border-radius: 3px;
  margin-right: 6px;
  flex-shrink: 0; /* Prevent swatch from shrinking */
}

.number-input {
  width: 60px; /* Reduced width */
  background-color: var(--input-bg-color);
  color: var(--input-text-color);
  border: 1px solid var(--input-border-color);
  border-radius: 3px;
  padding: 4px 0px;
  text-align: center;
  font-size: 0.85em;
  box-sizing: border-box;
  flex-shrink: 0; /* Prevent inputs from shrinking */
}

.gradient-display {
  flex-grow: 1;
  height: 24px; /* Slightly taller to accommodate select box styling */
  margin: 0 6px;
  border: 1px solid var(--input-border-color);
  border-radius: 3px;
  padding: 0 5px; /* Padding for select text */
  color: var(--input-text-color); /* Text color for the select box itself */
  -webkit-appearance: none; /* Remove default styling */
  -moz-appearance: none;
  appearance: none;
  background-size: cover; /* Ensure gradient covers the select */
  text-overflow: ellipsis; /* Add ellipsis for long colormap names */
  white-space: nowrap;
  overflow: hidden;
  cursor: pointer;
  line-height: 22px; /* Align text vertically */
}


/* Styling for options in dropdown */
.gradient-display option {
  background-color: var(--select-option-bg-color); 
  color: var(--select-option-text-color);           
  padding: 5px 8px;         
}


.action-icon {
  width: 22px;
  height: 22px;
  display: flex;
  align-items: center;
  justify-content: center;
  color: var(--action-icon-color);
  cursor: pointer;
  margin-left: 6px;
  font-size: 1.1em;
  flex-shrink: 0; /* Prevent icon from shrinking */
}

.action-icon:hover {
  color: var(--action-icon-hover-color);
}
/* Styles for Linear Transfer Function Editor */
.control-point {
  /* fill is now handled inline by :fill binding for selection */
  stroke: var(--border-color); /* Use general border color for consistency */
  stroke-width: 1;
  cursor: grab; /* Default cursor for hover */
}

.control-point.selected {
  /* fill is green via inline binding */
  stroke: #005000; /* Darker green stroke for selected point */
  stroke-width: 1.5px; /* Slightly thicker stroke */
}

.gaussian-control-point {
  fill: var(--action-icon-color);
  stroke: var(--border-color); /* Use general border color for consistency */
  stroke-width: 1;
  cursor: grab;
}

.gaussian-control-point.selected {
  fill: green;
  stroke: darkgreen;
  stroke-width: 2;
}

.gaussian-control-point:hover {
  fill: var(--action-icon-hover-color);
}

.gaussian-width-handle {
  stroke: var(--input-border-color);
  stroke-width: 1;
  cursor: ew-resize; /* East-West resize cursor for horizontal dragging */
}

.gaussian-width-handle:hover {
  fill: var(--action-icon-hover-color);
}
.linear-editor-svg,
.gaussian-editor-svg {
  display: block; /* Remove extra space below inline SVG */
  border: 1px solid var(--input-border-color);
  cursor: crosshair;
  /* background-color: var(--input-bg-color); */ /* Now set by inline style */
}

.svg-bg-rect {
  fill: transparent; /* Make background clickable but invisible */
}

.editor-line {
  stroke: var(--text-color);
  stroke-width: 2;
  pointer-events: none; /* Lines should not interfere with point interactions */
}

.editor-point {
  fill: var(--action-icon-color);
  stroke: var(--border-color);
  stroke-width: 1;
  cursor: grab;
}

.editor-point.selected {
  fill: var(--action-icon-hover-color);
  stroke: var(--text-color);
  stroke-width: 2;
}

.editor-point:hover {
  opacity: 0.8;
}

.picker-area-placeholder {
  display: flex;
  align-items: center;
  justify-content: center;
  height: var(--svgEditorHeight, 150px); /* Use variable or fallback */
  color: var(--main-picker-area-placeholder-text-color);
  font-style: italic;
  text-align: center;
  padding: 10px;
  box-sizing: border-box;
}

</style>
