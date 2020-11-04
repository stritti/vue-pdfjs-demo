<template>
  <div>
    <canvas ref="canvas" v-visible.once="drawPage" v-bind="canvasAttrs" class=".target"></canvas>
    <movable 
      class="testmove"
      ref="signature"
      v-if="canSign"
      :bounds="signatureBounds"
      :posTop="posTop"
      :posLeft="posLeft"
      :style="`width:${signatureDimensions.width}px; height:${signatureDimensions.height}px;`"
    ></movable>
  </div>
</template>

<script>
import debug from 'debug';

const log = debug('app:components/PDFPage');

import {PIXEL_RATIO} from '../utils/constants';
import visible from '../directives/visible';

export default {
  name: 'PDFPage',
  directives: {
    visible
  },

  props: {
    page: {
      type: Object, // instance of PDFPageProxy returned from pdf.getPage
      required: true
    },
    scale: {
      type: Number,
      required: true
    },
    optimalScale: {
      type: Number,
      required: true
    },
    isPageFocused: {
      type: Boolean,
      default: false
    },
    isElementFocused: {
      type: Boolean,
      default: false
    }
  },
  
  data () {
    return {
      needsRefresh: false,
      signatureBounds: null,
      signatureDimensions: {
        width: 150,
        height: 150,
      },
      posLeft: 0,
      posTop: 0,
      pageOriginalWidth: 210,
      pageOriginalHeight: 297,
      signatureOriginalWidth: 40,
      signatureOriginalHeight: 20,
      signatureX: 160,
      signatureY: 267,
      pageToSign: 2
    }
  },
  
  computed: {
    actualSizeViewport () {
      return this.viewport.clone({scale: this.scale});
    },

    canvasStyle () {
      const {width: actualSizeWidth, height: actualSizeHeight} = this.actualSizeViewport;
      const [pixelWidth, pixelHeight] = [actualSizeWidth, actualSizeHeight]
          .map(dim => Math.ceil(dim / PIXEL_RATIO));
      return `width: ${pixelWidth}px; height: ${pixelHeight}px;`;
    },

    canvasAttrs () {
      let {width, height} = this.viewport;
      [width, height] = [width, height].map(dim => Math.ceil(dim));
      const style = this.canvasStyle;

      return {
        width,
        height,
        style,
        class: 'pdf-page box-shadow'
      };
    },

    pageNumber () {
      return this.page.pageNumber;
    },

    canSign() {
      return parseInt(this.pageNumber) === parseInt(this.pageToSign) && this.signatureBounds && !this.needsRefresh
    }
  },
  
  watch: {
    scale: 'updateVisibility',

    page (_newPage, oldPage) {
      this.destroyPage(oldPage);
    },

    isElementFocused (isElementFocused) {
      if (isElementFocused) this.focusPage();
    }
  },
  
  created () {
    // PDFPageProxy#getViewport
    // https://mozilla.github.io/pdf.js/api/draft/PDFPageProxy.html
    this.viewport = this.page.getViewport(this.optimalScale);
  },

  mounted () {
    log(`Page ${this.pageNumber} mounted`);
    // this.$refs.moveable.dragTarget = document.querySelector(".scrolling-page")

    // console.log('MOV >>', this.$refs.moveable)
    this.updateVisibility()
  },
  methods: {
    focusPage () {
      if (this.isPageFocused) return;

      this.$emit('page-focus', this.pageNumber);
    },

    drawPage () {
      if (this.renderTask) return;

      const {viewport} = this;

      const canvasContext = this.$refs.canvas.getContext('2d');
      const renderContext = {canvasContext, viewport};

      // PDFPageProxy#render
      // https://mozilla.github.io/pdf.js/api/draft/PDFPageProxy.html
      this.renderTask = this.page.render(renderContext);
      this.renderTask
          .then(() => {
            this.$emit('page-rendered', {
              page: this.page,
              text: `Rendered page ${this.pageNumber}`
            });
          })
          .catch(response => {
            this.destroyRenderTask();
            this.$emit('page-errored', {
              response,
              page: this.page,
              text: `Failed to render page ${this.pageNumber}`
            });
          });
    },

    updateVisibility () {
      this.$parent.$emit('update-visibility');

      this.updateSignatureParameters()
    },

    updateSignatureParameters() {
      this.$nextTick(() => {
        
        this.signatureDimensions = {
          width: this.convertUnitToPixel('x',this.signatureOriginalWidth),
          height: this.convertUnitToPixel('y',this.signatureOriginalHeight)
        }

        this.posLeft = this.$refs.canvas.offsetLeft
        this.posTop = this.$refs.canvas.offsetTop
        this.signatureBounds = {
          x: [
            0,
            this.$refs.canvas.offsetWidth - this.signatureDimensions.width
          ],
          y: [
            0,
            this.$refs.canvas.offsetHeight - this.signatureDimensions.height
          ]
        }
        this.needsRefresh = true
        this.$nextTick(() => {
          this.needsRefresh = false

          this.$nextTick(() => {
            this.posLeft = this.convertUnitToPixel('x',this.signatureX) + this.$refs.canvas.offsetLeft
            this.posTop = this.convertUnitToPixel('y',this.signatureY) + this.$refs.canvas.offsetTop
          })
        })
      })
    },

    convertUnitToPixel(type, value) {
      const pageWidth = parseInt(this.$refs.canvas.offsetWidth)
      const pageHeight = parseInt(this.$refs.canvas.offsetHeight)
      if(type === 'x') {
        return pageWidth * value / this.pageOriginalWidth
      }

      return pageHeight * value / this.pageOriginalHeight

    },

    destroyPage (page) {
      // PDFPageProxy#_destroy
      // https://mozilla.github.io/pdf.js/api/draft/PDFPageProxy.html
      if (page) page._destroy();

      this.destroyRenderTask();
    },

    destroyRenderTask () {
      if (!this.renderTask) return;

      // RenderTask#cancel
      // https://mozilla.github.io/pdf.js/api/draft/RenderTask.html
      this.renderTask.cancel();
      delete this.renderTask;
    }
  },
  
  beforeDestroy () {
    this.destroyPage(this.page);
  }
};
</script>
<style>
.pdf-page {
  display: block;
  margin: 0 auto;
}

.testmove {
  display: block;
  top:0;
  position: absolute;
  height: 150px;
  width: 150px;
  margin: 0!important;
  background-color: rgba(0, 110, 255, 0.603)!important;
  color: white;
}

</style>
