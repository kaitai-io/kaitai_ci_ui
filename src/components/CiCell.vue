<template>
  <td v-bind:class="cssClassObject" @click="details = !details">
    {{ data.status || "unknown" }}
    <div class="add-info" v-if="details && hasDetails" v-on:click.stop>
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
      if (status === 'passed' && isKst) {
        return 'passed-kst';
      } else if (['passed', 'failed', 'leak'].indexOf(status) > -1) {
        return status;
      } else if (status === undefined) {
        return 'no-spec';
      }
    },
  },
};
</script>

<style scoped>
.has-details {
  cursor: pointer;
}

.passed {
  background: #dff0d8;
}

.passed-kst {
  background: #82dc75;
}

.skipped {
  background: #b9cfbb;
}

.failed {
  background: #f2dede;
}

.leak, .format-build-failed, .spec-build-failed {
  background: #f2cece;
}

.no-run, .no-spec {
  background: #f2bebe;
}

th.section {
  text-align: center;
  font-size: 200%;
}

.add-info {
  position: absolute;
  background: #fff;
  border: 1px solid #A0A0A0;
  cursor: auto;
  box-shadow: rgba(238, 238, 238, 0.5) 0 0 16px 16px;
}

.add-info .result:nth-child(even) .failure-info {
  background: #eee;
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
