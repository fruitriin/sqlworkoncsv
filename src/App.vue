<template>
  <div id="app">
    <h2>CSV</h2>
    <div class="csvimporter">
      <div v-for="csv in csvs">
        table name
        <input type="text" v-model="csv.name" value=""><br>
        <textarea cols="30" rows="10" v-on:blur="parseCsv(csv)" v-model="csv.source">

      </textarea>
      </div>
    </div>
    <h2>SQL</h2>
    <div class="sqlexecuter">
      <textarea class="js_sqlexecute" v-model="query" cols="30" rows="10"></textarea>
      <button v-on:click="executeSql(query)">execue SQL</button>
    </div>
    <h2>Output</h2>
    <div class="output">
      <h3>CSV</h3>
      <textarea v-model="resCSV" cols="30" rows="10"></textarea>
      <h3>JSON</h3>
      <textarea v-model="resJSON" cols="30" rows="10"></textarea>
      <h3>HTML TABLE</h3>
      <table border>
        <tr>
          <th v-for="(row , head) in resHTML[0]">{{head}}</th>
        </tr>
        <tr v-for="row in resHTML">
          <td v-for="td in row">
            {{td}}
          </td>
        </tr>
      </table>
    </div>
    </div>
  </div>
</template>

<script>
import Csv from './components/Csv.vue';
import Papa from "papaparse";

var db = openDatabase("sqlworkoncsv", "" , "SQL work on CSV", 1000);
var data;

export default {
  name: 'app',
  data:function(){
      var init = {
          name : "db1",
          source: ""

      }

      var csvs = new Array();
      data = {
          csvs: csvs,
              query: "",
          resCSV : "",
          resJSON: "",
          resHTML: "",
      }
      csvs.push(init);
      return data;
  },
  methods:{
    parseCsv : function(csv){
       var tmpparsed = Papa.parse(csv.source );
        if(tmpparsed.errors.length == 0){
            csv.parsed = tmpparsed.data;
            db.transaction(
                function(tr){
                  tr.executeSql(`DROP TABLE IF EXISTS ${csv.name}`,[],
                      function(){console.log(`DROP ${csv.name} TABLE`)},
                      function(){console.log(`Failed drop ${csv.name}`)}
                      );
                  console.log(csv.parsed)
                  tr.executeSql(`CREATE TABLE ${csv.name} ( ${csv.parsed[0].join(", ")} )`,[],
                      function(){console.log(`create ${csv.name } table`)},
                      function(){console.log(`failed to create ${csv.name} table `)}
                      );
                  csv.parsed.forEach(function(e,i,a){
                      if(i != 0){
                          tr.executeSql(`INSERT INTO ${csv.name} VALUES (${getInsertString(e)})`,[],null,
                            function(){console.log(`insert row failed`)}
                          );
                      }
                  });

                }
            )
        }

    },
    executeSql : function(query){
      db.transaction(function(tr){
          tr.executeSql(query,[],function(rt,rs){
              console.log(rt);
              console.log(`SUCCESS to execute Query: ${query}`);

              var tmpcsv =new Array();
              var tmpHTML = "<table>";
              for(var i = 0; i< rs.rows.length;  i++){
                  tmpcsv.push(rs.rows.item(i));
              }


              console.log(tmpcsv);

              data.resCSV = Papa.unparse(tmpcsv);
              data.resJSON = JSON.stringify(tmpcsv);
              data.resHTML =tmpcsv;
          },
          function(){
              console.log(`Failed to execute Query: ${query}`)
          });
      })
    }

  },

}

function getInsertString(element){
  let string = "";
  element.forEach(function(e){
      if(isNumber(e)){
          string += `${e.trim()} ,`;
      }else{
          string += `'${e.trim()}' ,`
      }
  })
  return string.substr(0,string.length-1);
}

var isNumber = function isNumber(value) {
    return typeof value === 'number' && isFinite(value);
};

</script>

<style>
#app {

}
</style>
