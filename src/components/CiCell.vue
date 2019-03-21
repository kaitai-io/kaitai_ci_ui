<template>
  <td v-bind:class="classObject">{{ data.status || "unknown" }}</td>
</template>

<script>
export default {
  name: 'CiCell',
  props: {
    data: Object,
  },
  computed: {
    classObject: function() {
      return {
        "passed": this.data.status == 'passed',
        "failed": this.data.status == 'failed',
        "leak": this.data.status == 'leak',
        "no-spec": this.data.status === undefined,
      };
    }
  }
}
</script>

<style scoped>
        .passed {
            background: #dff0d8;
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
            display: none;
            background: #F0F0F0;
            border: 1px solid #A0A0A0;
        }
</style>
