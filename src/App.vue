<template>
  <div id="app" class="container-fluid">
    <h1>Kaitai Struct CI</h1>

    <form id="search">
      <div class="row g-3">
        <div class="col-md-4">
          <input name="filterTest" class="form-control" v-model="filterTest" placeholder="Search test...">
        </div>
        <div class="col-md-4">
          <input name="filterTarget" class="form-control" v-model="filterTarget" placeholder="Search target...">
        </div>
        <div class="col-md-4 options-column">
          <div class="form-check form-check-inline">
            <input type="checkbox" class="form-check-input" v-model="skipPassed" id="only-failures-checkbox">
            <label class="form-check-label" for="only-failures-checkbox">
              Only failures
            </label>
          </div>
          <div class="form-check form-check-inline">
            <input type="checkbox" class="form-check-input" v-model="groupByLang" id="group-by-lang">
            <label class="form-check-label" for="group-by-lang">
              Group columns by language
            </label>
          </div>
        </div>
      </div>
    </form>

    <ci-grid
        :data="groupedGridData"
        :columns="groupedGridColumns"
        :meta="groupedGridMeta"
        :filter-key="filterTest"
        :filter-columns-key="filterTarget"
        :skip-passed="skipPassed"
        :group-by-lang="groupByLang">
    </ci-grid>
  </div>
</template>

<style>
.row > .col,
.row > [class*="col-"] {
  padding-right: 5px;
  padding-left: 5px;
}

.options-column {
  padding-left: 12px;
}

.form-check-inline {
  padding-top: 6px;
  padding-bottom: 6px;
}
</style>

<script>
import CiGrid from './components/CiGrid.vue';
import TARGET_PAIRS from './targetPairs.js';

export default {
  name: 'app',
  components: {
    CiGrid
  },
  data: function () {
    return {
      testData: {},
      filterTest: '',
      filterTarget: '',
      gridColumns: [],
      gridMeta: {},
      skipPassed: false,
      groupByLang: true,
    };
  },
  created: function () {
    const pairCmpFunc = this.getPairCompareFunc(TARGET_PAIRS);
    TARGET_PAIRS.forEach(pair => this.addOneJson(pair[0], pair[1], pairCmpFunc));
  },
  computed: {
    groupedGridColumns: function () {
      if (!this.groupByLang) {
        return this.gridColumns;
      }
      return Array.from(
          new Set(
              this.gridColumns.map(pair => pair.split('/')[0])
          )
      );
    },
    groupedGridData: function () {
      if (!this.groupByLang) {
        return Object.entries(this.testData)
            .map(([testName, testRow]) => ({name: testName, value: testRow}));
      }
      const rows = [];
      for (const testName in this.testData) {
        const testRow = this.testData[testName];
        const newTestRow = {};
        rows.push({name: testName, value: newTestRow});
        this.gridColumns.forEach(key => {
          let value = testRow[key];
          if (!value) {
            value = {};
          }
          if (!value.status) {
            value.status = 'unknown';
          }
          const pair = key.split('/');
          if (pair.length < 2) {
            console.error('target key "' + key + '" is invalid (does not have the format "{lang}/{variant}")');
          }
          const lang = pair[0];
          const variant = pair[1];

          if (!Object.prototype.hasOwnProperty.call(newTestRow, lang)) {
            newTestRow[lang] = {
              status: value.status,
              is_kst: true,
              failure: false,
              agg_results: [],
            };
          }
          const langData = newTestRow[lang];
          if (value.status !== langData.status) {
            langData.status = 'mixed';
          }
          if (value.status !== 'unknown' && !value.is_kst) {
            langData.is_kst = false;
          }
          if (value.failure) {
            langData.failure = true;
          }
          let mergeTo = langData.agg_results.find(res => {
            if (res.status !== value.status) {
              return false;
            }
            if (!res.failure && !value.failure) {
              return true;
            }
            if (res.failure && value.failure) {
              return (
                  res.failure.message === value.failure.message &&
                  res.failure.trace === value.failure.trace
              );
            }
            return false;
          });
          if (!mergeTo) {
            mergeTo = {
              status: value.status,
              variant_names: [],
            };
            if (value.failure) {
              mergeTo.failure = value.failure;
            }
            langData.agg_results.push(mergeTo);
          }
          mergeTo.variant_names.push(variant);
        });
      }
      return rows;
    },
    groupedGridMeta: function () {
      if (!this.groupByLang) {
        return this.gridMeta;
      }
      const newGridMeta = {};
      Object.entries(this.gridMeta).forEach(([pair, meta]) => {
        const lang = pair.split('/')[0];
        if (!Object.prototype.hasOwnProperty.call(newGridMeta, lang)) {
          newGridMeta[lang] = {
            lang: meta.lang,
            timestamp: meta.timestamp,
            ci: {},
            kst: 0,
            passed: 0,
          };
          return;
        }
        const langMeta = newGridMeta[lang];
        if (meta.timestamp > langMeta.timestamp) {
          langMeta.timestamp = meta.timestamp;
        }
      });
      this.groupedGridData.forEach(testRow => {
        Object.entries(testRow.value).forEach(([key, value]) => {
          if (!Object.prototype.hasOwnProperty.call(newGridMeta, key)) return;
          if (value.status === 'passed')
            newGridMeta[key].passed++;
          if (value.is_kst)
            newGridMeta[key].kst++;
        });
      });
      return newGridMeta;
    },
  },
  methods: {
    addOneJson: function (lang, version, pairCmpFunc) {
      var pair = lang + "/" + version;
      console.log("Querying data for", pair);
      fetch(
        "https://raw.githubusercontent.com/kaitai-io/ci_artifacts/" + pair + "/test_out/" + lang + "/ci.json"
      ).then(r => {
        if (!r.ok) {
          throw new Error("HTTP error, status code: " + r.status);
        }
        return r.json();
      }).then(json => {
        var meta = json.$meta;
        delete json.$meta;
        console.log("Got answer for " + pair + ", meta:", meta);

        // Aggregation
        var numPassed = 0;
        var numKst = 0;
        for (const testName in json) {
          if (!Object.prototype.hasOwnProperty.call(this.testData, testName)) {
            this.testData[testName] = {};
          }
          var row = this.testData[testName];
          delete json[testName]["name"];
          row[pair] = json[testName];
          if (row[pair].status === 'passed')
            numPassed++;
          if (row[pair].is_kst)
            numKst++;
        }

        // Generate output
        this.gridColumns.push(pair);
        this.gridColumns = this.gridColumns.sort(pairCmpFunc);

        meta.passed = numPassed;
        meta.kst = numKst;
        meta.timestamp = new Date(meta.timestamp);
        meta.artifactsUrl = "https://github.com/kaitai-io/ci_artifacts/tree/" + pair + "/test_out/" + lang;
        this.gridMeta[pair] = meta;
      }).catch(err => {
        console.warn("Cannot fetch data for " + pair + ". " + err);
      });
    },
    getPairCompareFunc: function (allPairs) {
      const findPairIdx = (val) => allPairs.findIndex(pair => pair.join('/') === val);
      return (a, b) => findPairIdx(a) - findPairIdx(b);
    },
  }
};
</script>
