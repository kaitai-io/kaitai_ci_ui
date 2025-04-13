<template>
  <td v-bind:class="cssClassObject" v-bind:style="mixedBgGradientStyle" @click="details = !details">
    {{ data.status || 'unknown' }}
    <div class="add-info position-absolute bg-white border" v-if="details && hasDetails" v-on:click.stop>
      <div class="result" v-for="(res, i) in results" :key="i">
        <h4 v-bind:class="[getCssClassByStatus(res.status, data.is_kst)]">
          <span v-if="res.variant_names">{{ res.variant_names.join(', ') }}: </span>
          <strong>{{ res.status }}</strong>
        </h4>
        <div class="failure-info" v-if="res.failure">
          <template v-if="res.failure.message">
            Message:
            <pre>{{ res.failure.message }}</pre>
          </template>
          <template v-if="res.failure.trace">
            Stack trace:
            <pre>{{ res.failure.trace }}</pre>
          </template>
        </div>
      </div>
    </div>
  </td>
</template>

<script>
const COLOR_BY_CLASS = Object.freeze({
  'passed': '#aea',
  'passed-kst': '#7d7',
  'skipped': '#cef',
  'failed': '#edd',
  'leak': '#ecc',
  'format-build-failed': '#fd9',
  'spec-build-failed': '#fd9',
  'no-spec': '#ebb',
});

export default {
  name: 'CiCell',
  props: {
    data: Object,
  },
  data: function () {
    return {"details": false};
  },
  computed: {
    results: function () {
      return this.data.agg_results || [this.data];
    },
    hasDetails: function () {
      return !!this.data.failure || this.data.status === 'mixed';
    },
    mixedBgGradientStyle: function () {
      if (this.data.status !== 'mixed') {
        return {};
      }
      const GRADIENT_SPACING = 10;
      const gradientStops = Array.from(
          new Set(this.data.agg_results.map(testRow => testRow.status)),
          (status, idx) => {
            const color = this.getStatusColorByCssClass(this.getCssClassByStatus(status, this.data.is_kst));
            return [
                color + ' ' + (idx * GRADIENT_SPACING).toFixed(0) + 'px',
                color + ' ' + ((idx + 1) * GRADIENT_SPACING).toFixed(0) + 'px',
            ];
          }
      ).flat();
      return {
        backgroundImage: 'repeating-linear-gradient(135deg, ' + gradientStops.join(', ') + ')',
      };
    },
    cssClassObject: function () {
      const classObj = {
        "has-details": this.hasDetails,
      };
      const cssClass = this.getCssClassByStatus(this.data.status, this.data.is_kst);
      if (cssClass) {
        classObj[cssClass] = true;
      }
      return classObj;
    },
  },
  methods: {
    getCssClassByStatus: function (status, isKst) {
      switch (status) {
        case 'passed':
          if (isKst) {
            return 'passed-kst';
          } else {
            return status;
          }
        case 'failed':
        case 'leak':
        case 'skipped':
          return status;
        case 'format_build_failed':
          return 'format-build-failed';
        case 'spec_build_failed':
          return 'spec-build-failed';
        case 'unknown':
        case undefined:
          return 'no-spec';
      }
    },
    getStatusColorByCssClass: function (cssClass) {
      return Object.prototype.hasOwnProperty.call(COLOR_BY_CLASS, cssClass) ?
          COLOR_BY_CLASS[cssClass] :
          'transparent';
    },
  },
};
</script>

<style scoped>
.has-details {
  cursor: pointer;
  position: relative;
}

.passed {
  background: #aea;
}

.passed-kst {
  background: #7d7;
}

.skipped {
  background: #cef;
}

.failed {
  background: #edd;
}

.leak {
  background: #ecc;
}

.format-build-failed, .spec-build-failed {
  background: #fd9;
}

.no-run, .no-spec {
  background: #ebb;
}

th.section {
  text-align: center;
  font-size: 200%;
}

.add-info {
  position: absolute;
  z-index: 100;
  cursor: auto;
  box-shadow: rgba(238, 238, 238, 0.5) 0 0 16px 16px;
}

.add-info:after {
  content: '';
  position: absolute;
  top: 100%;
  left: 100%;
  width: 24px;
  height: 24px;
}

.add-info pre {
  max-height: 48ex;
  overflow: auto;
}

.add-info .result h4,
.add-info .result .failure-info {
  padding: 1.5rem;
}

.add-info .result h4 {
  margin: 0;
}

.add-info .result .failure-info > :last-child {
  margin-bottom: 0;
}
</style>
