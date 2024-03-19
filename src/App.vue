<template>
  <div id="app" class="container-fluid">
    <h1>Kaitai Struct CI</h1>

    <form id="search">
      <div class="form-row">
        <div class="form-group col-md-4">
          <input name="filterTest" class="form-control" v-model="filterTest" placeholder="Search test...">
        </div>
        <div class="form-group col-md-4">
          <input name="filterTarget" class="form-control" v-model="filterTarget" placeholder="Search target...">
        </div>
        <div class="col-md-4">
          <div class="options-column">
            <label class="checkbox-inline" for="only-failures-checkbox">
              <input type="checkbox" v-model="skipPassed" id="only-failures-checkbox">
              Only failures
            </label>
            <label class="checkbox-inline" for="group-by-lang">
              <input type="checkbox" v-model="groupByLang" id="group-by-lang">
              Group columns by language
            </label>
          </div>
          <div class="options-column">
            <label class="radio-inline" for="suite-readonly">
              <input type="radio" name="suite" id="suite-readonly" value="readonly" v-model="suite">
              Parsing
            </label>
            <label class="radio-inline" for="suite-readwrite">
              <input type="radio" name="suite" id="suite-readwrite" value="readwrite" v-model="suite">
              Serialization
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
.form-row > .col,
.form-row > [class*="col-"] {
  padding-right: 5px;
  padding-left: 5px;
}

.form-row .options-column {
  display: inline-block;
  padding-left: 12px;
}

.col-md-4 label.checkbox-inline, .col-md-4 label.radio-inline {
  padding-top: 6px;
  padding-bottom: 6px;
}
</style>

<script>
import CiGrid from './components/CiGrid.vue';
import TARGET_PAIRS from './targetPairs.js';

const TARGET_LANGS_READWRITE = new Set([
  "java",
  "python",
]);
Object.freeze(TARGET_LANGS_READWRITE);

export default {
  name: 'app',
  components: {
    CiGrid
  },
  data: function () {
    return {
      filterTest: '',
      filterTarget: '',
      suiteData: {
        readonly: {
          testData: {},
          gridColumns: [],
          gridMeta: {},
        },
        readwrite: {
          testData: {},
          gridColumns: [],
          gridMeta: {},
        },
      },
      skipPassed: false,
      groupByLang: true,
      suite: 'readonly',
    };
  },
  created: function () {
    const pairCmpFunc = this.getPairCompareFunc(TARGET_PAIRS);
    TARGET_PAIRS.forEach(pair => {
      this.addOneJson(pair[0], pair[1], pairCmpFunc, 'readonly');
      if (TARGET_LANGS_READWRITE.has(pair[0])) {
        this.addOneJson(pair[0], pair[1], pairCmpFunc, 'readwrite');
      }
    });
  },
  computed: {
    groupedGridColumns: function () {
      const gridColumns = this.suiteData[this.suite].gridColumns;
      if (!this.groupByLang) {
        return gridColumns;
      }
      return Array.from(
          new Set(
              gridColumns.map(pair => pair.split('/')[0])
          )
      );
    },
    groupedGridData: function () {
      const suiteData = this.suiteData[this.suite];
      if (!this.groupByLang) {
        return Object.entries(suiteData.testData)
            .map(([testName, testRow]) => ({name: testName, value: testRow}));
      }
      const rows = [];
      for (const testName in suiteData.testData) {
        const testRow = suiteData.testData[testName];
        const newTestRow = {};
        rows.push({name: testName, value: newTestRow});
        suiteData.gridColumns.forEach(key => {
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
      const gridMeta = this.suiteData[this.suite].gridMeta;
      if (!this.groupByLang) {
        return gridMeta;
      }
      const newGridMeta = {};
      Object.entries(gridMeta).forEach(([pair, meta]) => {
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
    addOneJson: function (lang, version, pairCmpFunc, suite) {
      var pair = lang + "/" + version;
      lang += (suite === 'readwrite' ? "-write" : "");
      const suiteData = this.suiteData[suite];
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
          if (!Object.prototype.hasOwnProperty.call(suiteData.testData, testName)) {
            suiteData.testData[testName] = {};
          }
          var row = suiteData.testData[testName];
          row[pair] = json[testName];
          if (row[pair].status === 'passed')
            numPassed++;
          if (row[pair].is_kst)
            numKst++;
        }

        // Generate output
        suiteData.gridColumns.push(pair);
        suiteData.gridColumns.sort(pairCmpFunc);

        meta.passed = numPassed;
        meta.kst = numKst;
        meta.timestamp = new Date(meta.timestamp);
        meta.artifactsUrl = "https://github.com/kaitai-io/ci_artifacts/tree/" + pair + "/test_out/" + lang;
        suiteData.gridMeta[pair] = meta;
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
