<template>
  <td v-bind:class="cssClassObject" @click="details = !details">
    {{ data.status || "unknown" }}
    <div class="add-info" v-if="details && data.failure">
      <p>
        Message: {{data.failure.message}}
      </p>
      <pre v-if="data.failure.trace">{{data.failure.trace}}</pre>
    </div>
  </td>
</template>

<script>
export default {
  name: 'CiCell',
  props: {
    data: Object,
  },
  data: function() {
    return {"details": false};
  },
  computed: {
    cssClassObject: function() {
      return {
        "passed-kst": this.data.status == 'passed' && this.data.is_kst,
        "passed": this.data.status == 'passed' && !this.data.is_kst,
        "failed": this.data.status == 'failed',
        "leak": this.data.status == 'leak',
        "no-spec": this.data.status === undefined,
      };
    }
  },
}
</script>

<style scoped>
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
            cursor: pointer;
        }
        .leak, .format-build-failed, .spec-build-failed {
            background: #f2cece;
            cursor: pointer;
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
        }
</style>
