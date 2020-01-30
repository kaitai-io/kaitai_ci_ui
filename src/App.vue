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
         <div class="form-check col-md-4">
            <input type="checkbox" class="form-check-input" v-model="skipPassed" id="only-failures-checkbox">
            <label class="form-check-label" for="only-failures-checkbox">Only failures</label>
         </div>
      </div>
    </form>

    <ci-grid
      :data="gridData"
      :columns="gridColumns"
      :meta="gridMeta"
      :filter-key="filterTest"
      :filter-columns-key="filterTarget"
      :skip-passed="skipPassed">
    </ci-grid>
  </div>
</template>

<script>
import CiGrid from './components/CiGrid.vue'

export default {
  name: 'app',
  components: {
    CiGrid
  },
  data: function() {
    return {
      testData: {},
      filterTest: '',
      filterTarget: '',
      gridColumns: [],
      gridData: [],
      gridMeta: {},
      skipPassed: false,
    }
  },
  created: function () {
    var pairs = [
      ["cpp_stl_98", "gcc4.8_linux"],
      ["cpp_stl_98", "clang3.5_linux"],
      ["cpp_stl_98", "clang7.3_osx"],
      ["cpp_stl_98", "msvc141_windows_x64"],
      ["cpp_stl_11", "gcc4.8_linux"],
      ["cpp_stl_11", "clang3.5_linux"],
      ["cpp_stl_11", "clang7.3_osx"],
      ["cpp_stl_11", "msvc141_windows_x64"],
      ["csharp", "mono5.18.0"],
      ["csharp", "net45_windows"],
      ["graphviz", "2.36"],
      ["go", "1.10"],
      ["java", "openjdk7"],
      ["java", "oraclejdk8"],
      ["javascript", "nodejs4"],
      ["javascript", "nodejs7"],
      ["lua", "5.3"],
      ["nim", "1.0.0"],
      ["perl", "5.24"],
      ["php", "7.1"],
      ["python", "2.7"],
      ["python", "3.4"],
      ["python", "3.8"],
      ["ruby", "1.9"],
      ["ruby", "2.3"],
    ];
    for (var i in pairs) {
      this.addOneJson(pairs[i][0], pairs[i][1]);
    }
  },
  methods: {
    addOneJson: function (lang, version) {
      var pair = lang + "/" + version;
      console.log("Querying data for", lang, version, pair);
      fetch("https://raw.githubusercontent.com/kaitai-io/ci_artifacts/" + pair + "/test_out/" + lang + "/ci.json").then(r =>
        r.json()
      ).then(json => {
        var meta = json.$meta;
        delete json['$meta'];
        console.log("Got answer for", pair, ", meta:", meta);

        // Aggregation
        var numPassed = 0;
        var numKst = 0;
        for (var testName in json) {
          var row = this.testData[testName] || {"name": testName}
          delete json[testName]["name"];
          row[pair] = json[testName];
          if (row[pair].status === 'passed')
            numPassed++;
          if (row[pair].is_kst)
            numKst++;
          this.testData[testName] = row;
        }

        // Generate output
        this.gridColumns.push(pair);
        this.gridColumns = this.gridColumns.sort();
        this.gridData = [];
        for (var testName in this.testData) {
          this.gridData.push(this.testData[testName]);
        }

        meta.passed = numPassed;
        meta.kst = numKst;
        meta.timestamp = new Date(meta.timestamp);
        meta.artifactsUrl = "https://github.com/kaitai-io/ci_artifacts/tree/" + pair + "/test_out/" + lang;
        this.gridMeta[pair] = meta;
      });
    }
  }
}
</script>
