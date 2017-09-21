<template>
  <div id="app">
    <h2>CSV</h2>
    <div class="csvimporter">
      <div v-for="csv in csvs">
        table name
        <input type="text" v-model="csv.name" value=""><br>
        <textarea cols="30" rows="10" v-on:blur="parseCsv(csv)" v-model="csv.source"></textarea>
        <p class="err" v-for="mes in csv.messages">{{mes}}</p>
      </div>
      <button v-on:click="addTable()">+add table</button>
    </div>
    <h2>SQL</h2>
    <div class="sqlexecuter">
      <textarea class="js_sqlexecute" v-model="query" cols="30" rows="10"></textarea>
      <button v-on:click="executeSql(query)">execue SQL</button>
      <p class="err" v-for="err in errmessages">{{err}}</p>
      <p class="text">{{message}}</p>
    </div>
    <h2>Output</h2>
    <div class="output">
      <h3>CSV</h3>
      <textarea v-model="resCSV" cols="30" rows="10" readonly></textarea>
      <h3>JSON</h3>
      <textarea v-model="resJSON" cols="30" rows="10" readonly></textarea>
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
</template>

<script>
    import Papa from "papaparse";

    var db = openDatabase("sqlworkoncsv", "" , "SQL work on CSV", 1000);
    var data;

    export default {
        name: 'app',
        data:function(){
            var csvs = new Array();

            data = {
                csvs: [{
                    name : "db1",
                    source: `ID, name, text\n1,hoge,fuga`,
                    messages: new Array
                }],
                query: "select * from db1;",
                message: "",
                errmessages: [],
                resCSV : "",
                resJSON: "",
                resHTML: "",
            }

            parseTable();

            return data;
        },
        methods:{
            /**
             * csvをパースして、sqliteのテーブルとして格納
             * @param csv
             */
            parseCsv : function(csv){
                var tmpparsed = Papa.parse(csv.source );
                csv.messages = new Array();

                if(tmpparsed.errors.length != 0) {
                    csv.messages = tmpparsed.errors;
                    return;
                }

                csv.parsed = tmpparsed.data;
                db.transaction(
                    function (tr) {
                        tr.executeSql(`DROP TABLE IF EXISTS ${csv.name}`, [],
                            function () {
                                console.log(`DROP ${csv.name} TABLE`)
                            },
                            function () {
                                csv.messages.push(`Failed drop ${csv.name}`)
                            }
                        );
                        console.log(csv.parsed)
                        tr.executeSql(`CREATE TABLE ${csv.name} ( ${csv.parsed[0].join(", ")} )`, [],
                            function () {
                                console.log(`create ${csv.name } table`)
                            },
                            function () {
                                csv.messages.push(`failed to create ${csv.name} table `);
                            }
                        );
                        csv.parsed.forEach(function (e, i, a) {
                            if (i != 0) {
                                return;
                            }
                            tr.executeSql(`INSERT INTO ${csv.name} VALUES (${getInsertString(e)})`, [], null,
                                function () {
                                    csv.messages.push(`Failed insert row at ${i}`)
                                }
                            );
                        });

                    }
                )

            },
            /**
             * sqlの実行
             */
            executeSql : function(query){

                db.transaction(function(tr){
                    data.errmessages = new Array();
                    tr.executeSql(query,[],function(rt,rs){
                            var tmpcsv = new Array();
                            for(var i = 0; i< rs.rows.length;  i++){
                                tmpcsv.push(rs.rows.item(i));
                            }
                            data.resCSV = Papa.unparse(tmpcsv);
                            data.resJSON = JSON.stringify(tmpcsv);
                            data.resHTML = tmpcsv;
                        },
                        function(){
                            data.errmessages.push(`Failed to execute Query: ${query}`);
                       });
                })
            },
            /**
             * テーブルの追加
             */
            addTable: function(){
                data.csvs.push({
                    name : `db${data.csvs.length+1}`,
                    source: `ID, name, text\n1,hoge,fuga`,
                    messages: new Array
                });
                this.parseCsv(data.csvs[data.csvs.length-1]);
            }

        },

    }

    function getInsertString(element){
        return element.map((e) => {
                return isNumber(e) ? e.trim():`'${e.trim()}' `
        }).join(",");
    }

    var isNumber = function isNumber(value) {
        return typeof value === 'number' && isFinite(value);
    };

    function parseTable(){
        var tableNames = new Array;
        db.transaction((tr) => {
            const query = 'select name from sqlite_master where type="table" and name != "__WebKitDatabaseInfoTable__" order by name';
            //テーブル名一覧を取得
            tr.executeSql(query,[],(rt,rs) => {
                for(let i =0; i< rs.rows.length; i++ ){
                    let row = rs.rows.item(i)
                    console.log(row)
                    tableNames.push(row.name);
                }
                console.log(tableNames);
                //テーブル名一覧からテーブル一つずつ取得
                if (tableNames.length == 0){
                    return;
                }

                data.csvs = new Array;
                tableNames.forEach((table) => {
                    tr.executeSql(`select * from ${table}`, [], function (rt, rs) {
                        let soruce = [];
                        for (let i = 0; i < rs.rows.length; i++) {
                            soruce.push(rs.rows.item(i));
                        }

                        data.csvs.push({
                            name: table,
                            source: Papa.unparse(soruce),
                            messages: []
                        })
                    })
                })
            });

        })
    }

</script>

<style>
  #app {

  }

  .err {
    color: red;
    font-size: small;

  }
</style>
