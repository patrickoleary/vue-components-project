<template>
  <div class="advanced-color-picker" :class="[currentTheme + '-theme']">
    <div class="color-input-row">
      <input type="color" class="compact-color-picker" v-model="backgroundColorValue" title="Selected Base Color" />
      <input type="color" id="histogram-color-input" class="compact-color-picker" v-model="histogramColor" title="Histogram Color">
      <input type="text" class="number-input" v-model="minValue" title="Minimum Value" />
      <select class="gradient-display" v-model="selectedColormapName" :style="{ background: selectedColormapGeneratedGradient, color: 'transparent' }" title="Select Colormap">
        <option v-for="cmap in colormaps" :key="cmap.name" :value="cmap.name">
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
        <!-- Skew Indicator Line for selected Gaussian -->
        <line
          v-if="selectedGaussianId !== null && selectedGaussianSkewIndicatorLine && selectedGaussianSkewIndicatorLine.visible"
          :x1="selectedGaussianSkewIndicatorLine.x1"
          :y1="selectedGaussianSkewIndicatorLine.y1"
          :x2="selectedGaussianSkewIndicatorLine.x2"
          :y2="selectedGaussianSkewIndicatorLine.y2"
          :stroke="selectedGaussianSkewIndicatorLine.stroke"
          :stroke-width="selectedGaussianSkewIndicatorLine.strokeWidth"
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

        <!-- Gaussian Bias Handles -->
        <g v-if="selectedGaussianId !== null">
          <path
            v-for="handle in selectedGaussianBiasHandles"
            :key="handle.id"
            :d="handle.path"
            :fill="handle.fill"
            class="gaussian-bias-handle" 
            @mousedown.stop="handleBiasMouseDown($event, handle.gaussianId, handle.direction)"
          />
        </g>

        <!-- Bias Indicator Line for selected Gaussian -->
        <line
          v-if="selectedGaussianId !== null && selectedGaussianBiasIndicatorLine && selectedGaussianBiasIndicatorLine.visible"
          :x1="selectedGaussianBiasIndicatorLine.x1"
          :y1="selectedGaussianBiasIndicatorLine.y1"
          :x2="selectedGaussianBiasIndicatorLine.x2"
          :y2="selectedGaussianBiasIndicatorLine.y2"
          :stroke="selectedGaussianBiasIndicatorLine.stroke"
          :stroke-width="selectedGaussianBiasIndicatorLine.strokeWidth"
        />

        <!-- Gaussian Bias Handles -->
        <g v-for="handle in selectedGaussianBiasHandles" :key="handle.id">
          <path
            :d="handle.path"
            :fill="handle.fill"
            stroke="currentColor"
            stroke-width="1"
            :class="handle.class" 
            @mousedown.prevent.stop="handleBiasMouseDown($event, handle.gaussianId, handle.direction)"
            :title="handle.direction === 'up' ? 'Increase Bias' : 'Decrease Bias'"
          />
        </g>

        <!-- Gaussian Skew Handles -->
        <g v-for="item_skew in renderedGaussians" :key="'skew-handles-' + item_skew.id">
          <template v-if="item_skew.id === selectedGaussianId">
            <!-- Left Skew Handle (Triangle pointing left) -->
            <path
              :d="`M ${item_skew.controlPoint.x - 14},${item_skew.controlPoint.y} L ${item_skew.controlPoint.x - 8},${item_skew.controlPoint.y - 4} L ${item_skew.controlPoint.x - 8},${item_skew.controlPoint.y + 4} Z`"
              class="gaussian-skew-handle skew-left"
              :fill="currentTheme === 'light' ? '#4CAF50' : '#81C784'"
              stroke="currentColor"
              stroke-width="1"
              style="cursor: pointer;"
              @mousedown.prevent.stop="handleSkewMouseDown($event, item_skew.id, 'left')"
              title="Skew Left"
            />
            <!-- Right Skew Handle (Triangle pointing right) -->
            <path
              :d="`M ${item_skew.controlPoint.x + 14},${item_skew.controlPoint.y} L ${item_skew.controlPoint.x + 8},${item_skew.controlPoint.y - 4} L ${item_skew.controlPoint.x + 8},${item_skew.controlPoint.y + 4} Z`"
              class="gaussian-skew-handle skew-right"
              :fill="currentTheme === 'light' ? '#4CAF50' : '#81C784'"
              stroke="currentColor"
              stroke-width="1"
              style="cursor: pointer;"
              @mousedown.prevent.stop="handleSkewMouseDown($event, item_skew.id, 'right')"
              title="Skew Right"
            />
          </template>
        </g>
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
  name: 'AdvancedTransferFunctionEditor',
  props: {
    initialMinValueProp: {
      type: [String, Number],
      default: '573.113'
    },
    initialMaxValueProp: {
      type: [String, Number],
      default: '886.808'
    },
    histogramDataProp: {
      type: Array,
      default: () => []
    },
    initialThemeProp: {
      type: String,
      default: 'light',
      validator: function (value) {
        return ['light', 'dark'].includes(value);
      }
    },
    initialTransferFunctionModeProp: {
      type: String,
      default: 'linear',
      validator: function (value) {
        return ['linear', 'gaussian'].includes(value);
      }
    }
  },
  data() {
    return {
      backgroundColorValue: '#D3D3D3', // Default to light grey
      minValue: this.initialMinValueProp,
      maxValue: this.initialMaxValueProp,
      initialMinValue: this.initialMinValueProp, // For reset, now sourced from prop
      initialMaxValue: this.initialMaxValueProp, // For reset, now sourced from prop
      colormaps: colormapOptions,
      selectedColormapName: colormapOptions.length > 0 ? colormapOptions[0].name : null,
      transferFunctionMode: this.initialTransferFunctionModeProp,
      transferFunctionModes: [
        { value: 'linear', text: 'Linear' },
        { value: 'gaussian', text: 'Gaussian' }
      ],
      transferFunctionOpacity: 100,
      showHistogram: true, // Remains internally managed
      histogramColor: '#A9A9A9', // Remains internally managed
      currentTheme: this.initialThemeProp,
      // Linear Transfer Function Editor Data
      linearPoints: [{ x: 0, y: 0 }, { x: 1, y: 1 }],
      selectedPointIndex: null,
      draggingPointIndex: null,
      draggedPointObject: null,
      isDragging: false,
      mouseDownTimeout: null,
      pendingDragIndex: null,
      initialMousePosition: { x: 0, y: 0 },
      dragThreshold: 5,
      pointInteractionRecentlyActive: false,
      svgEditorWidth: 338,
      svgEditorHeight: 150,
      histogramData: [...this.histogramDataProp], // Initialize from prop
      histogramBins: 256,
      useLogScale: false,
      // Gaussian Transfer Function Editor Data
      gaussians: [],
      selectedGaussianId: null,
      draggingGaussianId: null,
      gaussianDragStartX: 0,
      originalGaussianW: 0,
      originalGaussianSkew: 0,
      initialMouseXForSkew: 0,
      initialMouseYForBias: 0,
      originalGaussianBias: 0,
      ignoreNextBackgroundClickForGaussian: false,
      nextGaussianId: 0,
      // New properties for width handles:
      activeDragMode: null,
      draggingWidthHandleSide: null,
      widthHandleInteractionActive: false,
      widthHandleSize: 12,
      biasInteractionActive: false,
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
    if (!this.histogramDataProp || this.histogramDataProp.length === 0) {
      this.generateSampleHistogram();
    }
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
      const leftHandleCenterNorm = { x: c - w, y: 0.05 }; // Position handles at y=0.05 (normalized)
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
      const rightHandleCenterNorm = { x: c + w, y: 0.05 }; // Position handles at y=0.05 (normalized)
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
                const s = g.skew || 0; // Current skew value, default to 0
                let gamma = 1;
                if (s > 1e-6) { // Positive skew (skewed to the right)
                  gamma = 1 + s;
                } else if (s < -1e-6) { // Negative skew (skewed to the left)
                  gamma = 1 / (1 - s); // s is negative, so (1-s) is > 1
                }

                const distFromCenter = x_norm - g.c;
                let effective_dist_for_exponent;
                if (distFromCenter >= 0) { // Point is to the right of or at the center
                  effective_dist_for_exponent = distFromCenter / gamma;
                } else { // Point is to the left of the center
                  effective_dist_for_exponent = distFromCenter * gamma;
                }

                // Gaussian equation for y_norm (0 at top, 1 at bottom)
                // Use baseSigma directly; skew effect is handled by adjusting distance via gamma
                // New logic: if x_norm is outside g.w from center g.c, y_norm_at_x = g.b.
                // Otherwise, use the full Gaussian formula.
                let y_norm_at_x;
                if (Math.abs(x_norm - g.c) > g.w) { // Changed (g.w / 2) to g.w
                  y_norm_at_x = 0.0;
                } else {
                  // baseSigma and effective_dist_for_exponent are already calculated immediately above this block.
                  y_norm_at_x = g.b + (g.h_control - g.b) * Math.exp(-Math.pow(effective_dist_for_exponent, 2) / (2 * Math.pow(baseSigma, 2)));
                }
                // Clamp y_norm_at_x to not exceed g.h_control
                if (y_norm_at_x > g.h_control) {
                  y_norm_at_x = g.h_control;
                }
                
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
          w: g.w,
          opacity: g.opacity,
          skew: g.skew
        });
      });
      return items;
    },

    selectedGaussianSkewIndicatorLine() {
      if (this.selectedGaussianId === null || typeof this.selectedGaussianId === 'undefined') {
        return { visible: false };
      }
      const gaussian = this.gaussians.find(g => g.id === this.selectedGaussianId);
      if (!gaussian) {
        return { visible: false };
      }

      const s = gaussian.skew || 0;
      let gamma = 1;
      if (s > 1e-6) { // Positive skew (skewed to the right)
        gamma = 1 + s;
      } else if (s < -1e-6) { // Negative skew (skewed to the left)
        gamma = 1 / (1 - s); // s is negative, so (1-s) is > 1, making gamma < 1 for F&S formula application
      }

      const baseHalfWidth = gaussian.w / 2;

      // Calculate the normalized X-coordinate for the center of the skewed distribution's span
      const normX_mid_skewed = gaussian.c + (baseHalfWidth / 2) * (gamma - (1 / gamma));

      // Convert to pixel coordinates
      const pixelX = this.normalizedToPixel({ x: normX_mid_skewed, y: gaussian.h_control }).x;
      const pixelY_h_control = this.normalizedToPixel({ x: gaussian.c, y: gaussian.h_control }).y;

      const lineHalfHeight = 14; // For a 28px tall line, reflecting triangle horizontal span

      return {
        x1: pixelX,
        y1: pixelY_h_control - lineHalfHeight,
        x2: pixelX, // Same X for a vertical line
        y2: pixelY_h_control + lineHalfHeight,
        stroke: 'red',
        strokeWidth: 2.5, // Increased thickness
        visible: true
      };
    },
    selectedGaussianBiasHandles() {
      if (this.selectedGaussianId === null) return [];
      const gaussian = this.gaussians.find(g => g.id === this.selectedGaussianId);
      if (!gaussian) return [];

      const peakPixel = this.normalizedToPixel({ x: gaussian.c, y: gaussian.h_control });
      const cx = peakPixel.x;
      const cpy_peak = peakPixel.y;
      // New dimensions to match skew handles' visual prominence:
      // Skew: length 6 (horizontal), base 8 (vertical)
      // Bias: length 8 (vertical), base 6 (horizontal)
      const handleSize = 3; // For a 6px wide base (half-width)
      const handleOffset = 8; // Distance from Gaussian peak to base of triangle
      const handleLength = 8; // Vertical length of the triangle (tip to base)

      // Upper handle (points up)
      // Tip Y: cpy_peak - handleOffset - handleLength (higher on screen)
      // Base Y: cpy_peak - handleOffset
      const upTrianglePath = `M ${cx},${cpy_peak - handleOffset - handleLength} L ${cx - handleSize},${cpy_peak - handleOffset} L ${cx + handleSize},${cpy_peak - handleOffset} Z`;

      // Lower handle (points down)
      // Tip Y: cpy_peak + handleOffset + handleLength (lower on screen)
      // Base Y: cpy_peak + handleOffset
      const downTrianglePath = `M ${cx},${cpy_peak + handleOffset + handleLength} L ${cx - handleSize},${cpy_peak + handleOffset} L ${cx + handleSize},${cpy_peak + handleOffset} Z`;
      
      return [
        {
          id: 'bias-up-' + gaussian.id,
          path: upTrianglePath,
          gaussianId: gaussian.id,
          direction: 'up',
          fill: 'green',
          class: 'bias-handle'
        },
        {
          id: 'bias-down-' + gaussian.id,
          path: downTrianglePath,
          gaussianId: gaussian.id,
          direction: 'down',
          fill: 'green',
          class: 'bias-handle'
        }
      ];
    },
    selectedGaussianBiasIndicatorLine() {
      if (this.selectedGaussianId === null) return { visible: false };
      const gaussian = this.gaussians.find(g => g.id === this.selectedGaussianId);
      if (!gaussian) return { visible: false };

      const g_b = gaussian.b; // Gaussian's actual baseline parameter (0 to 1)
      const g_h_control = gaussian.h_control; // Gaussian's peak height (normalized 0 to 1)

      // Define constants for bias handle geometry (mirroring selectedGaussianBiasHandles)
      const biasHandleVerticalOffset = 10; // pixels
      const biasTriangleHeight = 6; // pixels

      // Calculate the Y-coordinate of the tip of the bottom bias triangle
      const peakPixelY = this.normalizedToPixel({ x: 0, y: g_h_control }).y;
      const bottomTriangleTipPixelY = peakPixelY + biasHandleVerticalOffset + biasTriangleHeight;
      const yNormBottomTriangleTip = this.pixelToNormalized({ x: 0, y: bottomTriangleTipPixelY }).y;

      // Interpolate the red line's normalized Y position
      // When g_b = 0, lineYNorm = g_h_control
      // When g_b = 1, lineYNorm = yNormBottomTriangleTip
      let lineYNorm = g_h_control + (yNormBottomTriangleTip - g_h_control) * g_b;

      // Clamp the line's Y position to be within the editor bounds [0, 1] (normalized)
      // Note: y=0 is bottom, y=1 is top for normalized, but pixel y is inverted.
      // Clamping to ensure it doesn't go out of SVG view if calculations are extreme.
      lineYNorm = Math.max(0, Math.min(1, lineYNorm)); 

      const lineYPixel = this.normalizedToPixel({ x: 0, y: lineYNorm }).y;
      const centerXPixel = this.normalizedToPixel({ x: gaussian.c, y: 0 }).x;
      const halfWidth = 14; // For a 28px wide line

      return {
        x1: centerXPixel - halfWidth,
        y1: lineYPixel,
        x2: centerXPixel + halfWidth,
        y2: lineYPixel,
        stroke: 'red',
        strokeWidth: 2.5,
        visible: true
      };    
    },
    selectedColormap() {
      if (!this.selectedColormapName || !this.colormaps) return null;
      return this.colormaps.find(cmap => cmap.name === this.selectedColormapName);
    },
    selectedColormapGeneratedGradient() {
      if (!this.selectedColormap || !this.selectedColormap.points) {
        return 'linear-gradient(to right, black, white)'; // Default/error
      }
      return this.generateGradientFromPoints(this.selectedColormap.points);
    },
    currentGradientStops() {
      if (!this.selectedColormap || !this.selectedColormap.points || this.selectedColormap.points.length === 0) {
        // Return a default black to white gradient if no valid points
        return [
          { offset: '0%', color: 'rgb(0,0,0)' },
          { offset: '100%', color: 'rgb(255,255,255)' }
        ];
      }
      // Sort points by x, just in case they aren't already.
      const sortedPoints = [...this.selectedColormap.points].sort((a, b) => a.x - b.x);

      return sortedPoints.map(point => {
        const r = Math.round(point.r * 255);
        const g = Math.round(point.g * 255);
        const b = Math.round(point.b * 255);
        return {
          offset: `${Math.round(point.x * 100)}%`,
          color: `rgb(${r},${g},${b})`
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
    // Clean up global event listeners for the Linear Transfer Function Editor
    window.removeEventListener('mousemove', this.handleEditorMouseMove);
    window.removeEventListener('mouseup', this.handleEditorMouseUp);

    // Clean up global event listeners for the Gaussian Transfer Function Editor
    // Center point dragging
    document.removeEventListener('mousemove', this.handleGaussianCenterPointMouseMove);
    document.removeEventListener('mouseup', this.handleGaussianDragMouseUp); // Also used by width dragging

    // Width handle dragging
    document.removeEventListener('mousemove', this.handleGaussianWidthHandleMouseMove);
    // mouseup is handleGaussianDragMouseUp

    // Skew handle dragging
    document.removeEventListener('mousemove', this.handleGaussianSkewMouseMove);
    document.removeEventListener('mouseup', this.handleGaussianSkewMouseUp);

    // Bias handle dragging
    document.removeEventListener('mousemove', this.handleGaussianBiasMouseMove);
    document.removeEventListener('mouseup', this.handleGaussianBiasMouseUp);
  },
  methods: {
    generateGradientFromPoints(pointsArray) {
      if (!pointsArray || pointsArray.length === 0) {
        return 'linear-gradient(to right, black, white)'; // Default or error gradient
      }
      // Sort points by x, just in case they aren't already.
      const sortedPoints = [...pointsArray].sort((a, b) => a.x - b.x);

      const stops = sortedPoints.map(point => {
        const r = Math.round(point.r * 255);
        const g = Math.round(point.g * 255);
        const b = Math.round(point.b * 255);
        const position = Math.round(point.x * 100);
        return `rgb(${r}, ${g}, ${b}) ${position}%`;
      });

      return `linear-gradient(to right, ${stops.join(', ')})`;
    },
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
            const s = g.skew || 0; // Current skew value, default to 0
            let gamma = 1;
            if (s > 1e-6) { // Positive skew (skewed to the right)
              gamma = 1 + s;
            } else if (s < -1e-6) { // Negative skew (skewed to the left)
              gamma = 1 / (1 - s); // s is negative, so (1-s) is > 1
            }
            // if s is very close to 0, gamma remains 1 (no skew)

            const distFromCenter = x_norm - g.c;
            let effective_dist_for_exponent;

            if (distFromCenter >= 0) { // Point is to the right of or at the center
              effective_dist_for_exponent = distFromCenter / gamma;
            } else { // Point is to the left of the center
              effective_dist_for_exponent = distFromCenter * gamma;
            }
            
            const amplitudeFactor = g.h_control - g.b; // h_control is peak, b is baseline
            // Use baseSigma directly; skew effect is handled by adjusting distance via gamma
            const exponent = -Math.pow(effective_dist_for_exponent, 2) / (2 * baseSigma * baseSigma);
            // New logic: if x_norm is outside g.w from center g.c, y_gaussian_at_x = g.b.
            // Otherwise, use the full Gaussian formula.
            if (Math.abs(x_norm - g.c) > g.w) { // Changed (g.w / 2) to g.w
              y_gaussian_at_x = 0.0;
            } else {
              // amplitudeFactor and exponent are already calculated immediately above this block.
              y_gaussian_at_x = g.b + amplitudeFactor * Math.exp(exponent);
            }
            // Clamp y_gaussian_at_x to not exceed g.h_control
            if (y_gaussian_at_x > g.h_control) {
              y_gaussian_at_x = g.h_control;
            }
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
        // Ensure the point still exists before sorting and finding index
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
  
  
  
  
      if (this.biasInteractionActive) {
    
    return; // Prevent if bias interaction is active
  }
  if (this.pointInteractionRecentlyActive || this.ignoreNextBackgroundClickForGaussian) {
    
    this.pointInteractionRecentlyActive = false; // Reset after checking
    this.ignoreNextBackgroundClickForGaussian = false; 
    return;
  }
  if (!this.$refs.gaussianEditorSvg) {
    console.log('[DEBUG] Early return: gaussianEditorSvg ref missing');
    return;
  }

  const clickCoords = this.getSvgCoordinates(event, this.$refs.gaussianEditorSvg);
  if (!clickCoords) {
    console.log('[DEBUG] Early return: clickCoords missing');
    return;
  }

      // Check if the click is near an existing control point
      for (const g of this.gaussians) {
    const pointPixel = this.normalizedToPixel({ x: g.c, y: g.h_control });
    const dx = clickCoords.x - pointPixel.x;
    const dy = clickCoords.y - pointPixel.y;
    const distSq = dx * dx + dy * dy;
    

    if (distSq <= GAUSSIAN_POINT_HIT_RADIUS_SQUARED) {
      
      this.selectedGaussianId = g.id;
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
    opacity: 1.0,                // Opacity (0-1, normalized)
    skew: 0                      // Skewness (e.g., -1 to 1, 0 for no skew)
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

    handleSkewMouseDown(event, gaussianId, side) {
      event.preventDefault();
      event.stopPropagation();

      this.draggingGaussianId = gaussianId;
      this.activeDragMode = side === 'left' ? 'skewLeft' : 'skewRight';

      const svg = this.$refs.gaussianEditorSvg;
      if (!svg) {
        console.error('Gaussian editor SVG element not found.');
        return;
      }

      const initialMousePos = this.getSvgCoordinates(event, svg);
      if (!initialMousePos) {
        console.error('Could not get SVG coordinates for skew mouse down.');
        return;
      }
      this.initialMouseXForSkew = initialMousePos.x;

      const gaussian = this.gaussians.find(g => g.id === gaussianId);
      if (gaussian) {
        if (side === 'right' && gaussian.skew < 0) {
          this.originalGaussianSkew = 0;
        } else if (side === 'left' && gaussian.skew > 0) {
          this.originalGaussianSkew = 0;
        } else {
          this.originalGaussianSkew = gaussian.skew;
        }
      } else {
        console.error(`Gaussian with ID ${gaussianId} not found for skew operation.`);
        this.activeDragMode = null; // Reset drag mode if Gaussian not found
        return;
      }

      // Add listeners for mouse move and mouse up to handle the dragging
      document.addEventListener('mousemove', this.handleGaussianSkewMouseMove);
      document.addEventListener('mouseup', this.handleGaussianSkewMouseUp);
    },

    handleGaussianSkewMouseMove(event) {
      if (this.activeDragMode !== 'skewLeft' && this.activeDragMode !== 'skewRight') {
        return;
      }
      event.preventDefault();

      const svg = this.$refs.gaussianEditorSvg;
      if (!svg) {
        console.error('Gaussian editor SVG element not found during skew move.');
        return;
      }

      const currentMousePos = this.getSvgCoordinates(event, svg);
      if (!currentMousePos) {
        console.error('Could not get SVG coordinates during skew move.');
        return;
      }

      const gaussian = this.gaussians.find(g => g.id === this.draggingGaussianId);
      if (!gaussian) {
        console.error(`Gaussian with ID ${this.draggingGaussianId} not found during skew move.`);
        this.handleGaussianSkewMouseUp(event); // Clean up
        return;
      }

      const deltaX = currentMousePos.x - this.initialMouseXForSkew;
      const skewSensitivity = 0.005; // Adjust for desired sensitivity
      const changeInSkew = deltaX * skewSensitivity;
      let newSkew;

      if (this.activeDragMode === 'skewRight') {
        newSkew = this.originalGaussianSkew + changeInSkew;
        newSkew = Math.max(0, Math.min(1, newSkew)); // Clamp to [0, 1]
      } else { // activeDragMode === 'skewLeft'
        newSkew = this.originalGaussianSkew + changeInSkew; // Note: if original was >0, it's now 0.
                                                          // Dragging left (deltaX < 0) will make changeInSkew negative.
        newSkew = Math.max(-1, Math.min(0, newSkew)); // Clamp to [-1, 0]
      }

      gaussian.skew = newSkew;
    },

    handleBiasMouseDown(event, gaussianId, direction) {
      event.preventDefault();
      event.stopPropagation();
      this.biasInteractionActive = true;
      this.ignoreNextBackgroundClickForGaussian = true; // Reinforce this
      this.draggingGaussianId = gaussianId;
      this.activeDragMode = 'bias'; // Using a general 'bias' mode
      const svg = this.$refs.gaussianEditorSvg;
      if (!svg) return;
      const initialMousePos = this.getSvgCoordinates(event, svg);
      this.initialMouseYForBias = initialMousePos.y;
      const gaussian = this.gaussians.find(g => g.id === gaussianId);
      if (gaussian) {
        this.originalGaussianBias = gaussian.b;
      }
      document.addEventListener('mousemove', this.handleGaussianBiasMouseMove);
      
      document.addEventListener('mouseup', this.handleGaussianBiasMouseUp);
    },
    handleGaussianBiasMouseMove(event) {
      if (this.activeDragMode !== 'bias') return;
      event.preventDefault();
      const svg = this.$refs.gaussianEditorSvg;
      if (!svg) return;
      const currentMousePos = this.getSvgCoordinates(event, svg);
      const gaussian = this.gaussians.find(g => g.id === this.draggingGaussianId);
      if (!gaussian) return;

      const deltaY_pixels = currentMousePos.y - this.initialMouseYForBias;
      
      // Sensitivity: 1.0 change in bias for dragging full editor height
      const biasSensitivity = 1.0 / this.svgEditorHeight; 
      const changeInBias = deltaY_pixels * biasSensitivity;
      
      let newBias = this.originalGaussianBias + changeInBias;
      newBias = Math.max(0, Math.min(1, newBias)); // Clamp bias to [0, 1]
      
      gaussian.b = newBias;
    },
    handleGaussianBiasMouseUp(event) {
  
  // Always remove listeners and reset flags, even if not in bias mode
  document.removeEventListener('mousemove', this.handleGaussianBiasMouseMove);
  
  document.removeEventListener('mouseup', this.handleGaussianBiasMouseUp);
  this.biasInteractionActive = false;
  
  this.ignoreNextBackgroundClickForGaussian = true;
  
  if (this.activeDragMode !== 'bias') return;
  this.activeDragMode = null;
  this.draggingGaussianId = null;
  // Call the generic drag mouse up handler to reset all shared state and listeners
  this.handleGaussianDragMouseUp(event);

},

    handleGaussianSkewMouseUp(event) {
      // Logic for finalizing skew adjustment and cleaning up listeners will go here.
      this.activeDragMode = null;
      this.draggingGaussianId = null;
      document.removeEventListener('mousemove', this.handleGaussianSkewMouseMove);
      document.removeEventListener('mouseup', this.handleGaussianSkewMouseUp);
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
  padding: 0; /* Remove padding for a more compact look */
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

.bias-handle {
  cursor: pointer; /* Pointing finger cursor for consistency */
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
