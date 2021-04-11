<template>
  <td v-bind:class="cssClassObject" @click="details = !details">
    {{ data.status || "unknown" }}
    <div class="add-info" v-if="details && hasDetails">
      <div class="variant">
        <h4><strong>{{ data.status }}</strong></h4>
        <template v-if="data.failure.message">
          Message:
          <pre>{{ data.failure.message }}</pre>
        </template>
        <template v-if="data.failure.trace">
          Stack trace:
          <pre>{{ data.failure.trace }}</pre>
        </template>
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
    hasDetails: function () {
      return !!this.data.failure;
    },
    cssClassObject: function () {
      return {
        "has-details": this.hasDetails,
        "passed-kst": this.data.status === 'passed' && this.data.is_kst,
        "passed": this.data.status === 'passed' && !this.data.is_kst,
        "failed": this.data.status === 'failed',
        "leak": this.data.status === 'leak',
        "no-spec": this.data.status === undefined,
      };
    }
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
  background: #F0F0F0;
  border: 1px solid #A0A0A0;
  padding: 1.5rem;
}

.add-info > :last-child,
.add-info > :last-child :last-child {
  margin-bottom: 0;
}
</style>
