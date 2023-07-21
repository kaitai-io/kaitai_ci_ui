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
        <div class="col-md-4 options-column">
          <label class="checkbox-inline" for="only-failures-checkbox">
            <input type="checkbox" v-model="skipPassed" id="only-failures-checkbox">
            Only failures
          </label>
          <label class="checkbox-inline" for="group-by-lang">
            <input type="checkbox" v-model="groupByLang" id="group-by-lang">
            Group columns by language
          </label>
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
  padding-left: 12px;
}

.col-md-4 label.checkbox-inline {
  padding-top: 6px;
  padding-bottom: 6px;
}
</style>

<script>
import CiGrid from './components/CiGrid.vue';

const TARGET_PAIRS = [
  ["cpp_stl_98", "clang3.4-linux-x86_64"],
  ["cpp_stl_98", "clang11-linux-x86_64"],
  ["cpp_stl_98", "gcc4.8-linux-x86_64"],
  ["cpp_stl_98", "gcc11-linux-x86_64"],
  ["cpp_stl_98", "clang14-macos-x86_64"],
  ["cpp_stl_98", "msvc141-windows-x64"],
  ["cpp_stl_11", "clang3.4-linux-x86_64"],
  ["cpp_stl_11", "clang11-linux-x86_64"],
  ["cpp_stl_11", "gcc4.8-linux-x86_64"],
  ["cpp_stl_11", "clang14-macos-x86_64"],
  ["cpp_stl_11", "msvc141-windows-x64"],
  ["csharp", "mono5.18.0"],
  ["csharp", "net45-windows-x64"],
  ["csharp", "netcore2.2.103-linux-x86_64"],
  ["csharp", "netcore3.0.100-linux-x86_64"],
  ["graphviz", "2.36"],
  ["go", "1.19-linux-x86_64"],
  ["java", "zulu7-linux-x86_64"],
  ["java", "temurin8-linux-x86_64"],
  ["java", "temurin11-linux-x86_64"],
  ["java", "temurin17-linux-x86_64"],
  ["javascript", "nodejs4-linux-x86_64"],
  ["javascript", "nodejs8-linux-x86_64"],
  ["javascript", "nodejs10-linux-x86_64"],
  ["javascript", "nodejs12-linux-x86_64"],
  ["javascript", "nodejs18-linux-x86_64"],
  ["lua", "5.3-linux-x86_64"],
  ["nim", "1.6.0-linux-x86_64"],
  ["perl", "5.24-linux-x86_64"],
  ["perl", "5.38-linux-x86_64"],
  ["php", "7.1-linux-x86_64"],
  ["php", "8.2-linux-x86_64"],
  ["python", "2.7-linux-x86_64"],
  ["python", "3.4-linux-x86_64"],
  ["python", "3.11-linux-x86_64"],
  ["construct", "python2.7-linux-x86_64"],
  ["construct", "python3.4-linux-x86_64"],
  ["construct", "python3.11-linux-x86_64"],
  ["ruby", "1.9-linux-x86_64"],
  ["ruby", "2.3-linux-x86_64"],
  ["ruby", "3.2-linux-x86_64"],
];
TARGET_PAIRS.forEach(pair => Object.freeze(pair));
Object.freeze(TARGET_PAIRS);

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
      gridData: [],
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
        return this.gridData;
      }
      const pairCmpFunc = this.getPairCompareFunc(TARGET_PAIRS);
      return this.gridData.map(testRow => {
        const newTestRow = {};
        const keys = Object.keys(testRow).sort(pairCmpFunc);
        keys.forEach((key) => {
          const value = testRow[key];
          if (key === 'name') {
            newTestRow[key] = value;
            return;
          }
          const pair = key.split('/', 2);
          if (pair.length < 2) {
            console.error('target key "' + key + '" is invalid (does not have the format "{lang}/{variant}")');
          }
          const lang = pair[0];
          const variant = pair[1];

          if (!Object.prototype.hasOwnProperty.call(newTestRow, lang)) {
            newTestRow[lang] = {
              status: value.status,
              is_kst: value.is_kst,
              failure: false,
              agg_status_set: new Set(),
              agg_results: [],
            };
          }
          const langData = newTestRow[lang];
          if (langData.status !== 'mixed' && value.status !== langData.status) {
            langData.status = 'mixed';
          }
          if (value.is_kst !== langData.is_kst) {
            langData.is_kst = false;
          }
          if (value.failure && !langData.failure) {
            langData.failure = true;
          }
          const mergeTo = langData.agg_results.find(res => {
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
          if (mergeTo) {
            mergeTo.variant_names.push(variant);
          } else {
            const res = {
              status: value.status,
              variant_names: [variant],
            };
            if (value.failure) {
              res.failure = value.failure;
            }
            langData.agg_results.push(res);
          }
          langData.agg_status_set.add(value.status);
        });
        return newTestRow;
      });
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
        Object.entries(testRow).forEach(([key, value]) => {
          if (key === 'name' || !Object.prototype.hasOwnProperty.call(newGridMeta, key)) return;
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
            this.testData[testName] = {"name": testName};
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
        this.gridData = [];
        for (const testName in this.testData) {
          this.gridData.push(this.testData[testName]);
        }

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
      return (a, b) => {
        const findPairIdx = (val) => allPairs.findIndex(pair => pair.join('/') === val);
        return findPairIdx(a) - findPairIdx(b);
      };
    },
  }
};
</script>
