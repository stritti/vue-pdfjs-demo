<template>
  <div>
    <canvas ref="canvas" v-visible.once="drawPage" v-bind="canvasAttrs" class=".target"></canvas>
    <movable class="testmove"
             v-if="signatureBounds && !needsRefresh"
             :bounds="signatureBounds"
             :posTop="signatureBounds.y[0]"
             :posLeft="signatureBounds.x[0]"><span>teste</span></movable>
  </div>
</template>

<script>
// import Moveable from 'vue-moveable'
import debug from 'debug';

const log = debug('app:components/PDFPage');

import {PIXEL_RATIO} from '../utils/constants';
import visible from '../directives/visible';

export default {
  name: 'PDFPage',
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

  directives: {
    visible
  },
  data () {
    return {
      needsRefresh: false,
      signatureBounds: null,
      signatureDimensions: {
        width: 150,
        height: 150,
      }
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
    }
  },

  methods: {
    handleDrag ({target, transform}) {
      console.log("onDrag", transform);
      target.style.transform = transform;
    },

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

      const originalWidth = 0.1
      const originalHeight = 0.05
      const pageWidth = parseInt(this.$refs.canvas.getAttribute('width'))
      const pageHeight = parseInt(this.$refs.canvas.getAttribute('height'))
      this.signatureDimensions = {
        width: pageWidth * this.scale * originalWidth,
        height: pageHeight * this.scale * originalHeight
      }

      this.signatureBounds = {
        x: [
          this.$refs.canvas.offsetLeft,
          this.$refs.canvas.offsetLeft + this.$refs.canvas.offsetWidth - 150
        ],
        y: [
          this.$refs.canvas.offsetTop,
          this.$refs.canvas.offsetTop + this.$refs.canvas.offsetHeight - 150
        ]
      }
      this.needsRefresh = true
      this.$nextTick(() => this.needsRefresh = false)
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
  background: #f00;
  color: white;
}

</style>
