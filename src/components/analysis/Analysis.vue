<template>

    <div>
        <!--    筛选框-->
        <div class="screen" style="display: flex;flex-direction: column;justify-content: space-around">

            <div id="source_data">
                <p style="margin-right: 20px">生源数据:</p>
                <div style="display:flex;align-items: center">
                    <el-upload action accept=".xlsx,.xls" :auto-upload="false" :show-file-list="false"
                    :on-change="handleChange">
                        <el-button plain size="small" style="height: 30px;">选择文件</el-button>
                    </el-upload>
                </div>
            </div>
            <div id="data_scale">
                <p style="margin-right: 20px">数据范围:</p>
                <div style="display: flex;align-items: center;">
                    <el-dropdown  size="small" split-button trigger="click" @command="handleIndexBegin">
                        {{rowbegin}}
                        <el-dropdown-menu slot="dropdown" id="dropdown_begin">
                            <el-dropdown-item v-for="(item,index) in bcols" :key="index" :command="item.page">{{item.page}}</el-dropdown-item>
                        </el-dropdown-menu>
                    </el-dropdown>
                    <p style="margin-right: 20px;margin-left: 20px">—</p>
                    <el-dropdown size="small" split-button trigger="click" @command="handleIndexEnd">
                          {{rowend}}
                        <el-dropdown-menu slot="dropdown" id="dropdown_end">
                            <el-dropdown-item v-for="(item,index) in bcols" :key="index" :command="item.page">{{item.page}}</el-dropdown-item>
                        </el-dropdown-menu>
                    </el-dropdown>
                </div>
            </div>
            <div id="field_map" style="display: flex;align-items: center">
                <p style="margin-right: 20px">字段映射:</p>
                <el-dropdown  size="small" split-button trigger="click" @command="handleSchField">
                    {{schfiled}}
                        <el-dropdown-menu slot="dropdown" id="school_filed">
                            <el-dropdown-item v-for="(item,index) in sch_fields" :key="index" :command="item.field">{{item.field}}</el-dropdown-item>
                        </el-dropdown-menu>
                </el-dropdown>
            </div>

            <div id="conditions">
                <el-checkbox-group v-model="checkboxGroup1" size="medium">
                    <el-checkbox label="地区"  border style="width: 100px;margin-right: 0px"></el-checkbox>
                    <el-checkbox label="985"  border style="width: 100px;margin-right: 0px"></el-checkbox>
                    <el-checkbox label="211"  border style="width: 100px"></el-checkbox>
                </el-checkbox-group>
            </div>
            <div style="margin-top: 15px;margin-bottom: 15px">
                <el-button type="primary" size="medium" id="finish" plain @click="getResult()">完成</el-button>
            </div>
        </div>
<!--        地图分析结果-->
        <div class="map" style="width: 100%;display: flex">
            <div id="region" style="width: 60%;height: 600px">
            </div>
            <div id="show" style="width: 40%;display: flex;flex-direction: column">
                <el-row style="display: flex;justify-content: space-around">
                    <el-button size="small" @click="exportEXCEL('xlsx')">导出为EXCEL</el-button>
                    <el-button size="small" @click="exportEXCEL('csv')">导出为CSV</el-button>
                    <el-button size="small" @click="printparams()">打印</el-button>
                </el-row>
                <el-table :data="mapresult" stripe style="width: 100%">
                    <el-table-column label="地区"  prop="name" align="center"></el-table-column>
                    <el-table-column label="人数"  prop="value" align="center"></el-table-column>
                </el-table>
            </div>
        </div>
<!--其他数据分析-->
        <div class="other_data">
            <div id="title">
                <p>其他数据</p>
            </div>
            <div id="other_result" style="width: 100%;display: flex">
                <div style="width: 70%;margin-right: 50px">
                    <el-table :data="otherdata" stripe>
                        <el-table-column label="字段"  prop="depend" align="center"></el-table-column>
                        <el-table-column label="人数"  prop="count" align="center"></el-table-column>
                        <el-table-column label="比例"  prop="ratio" align="center"></el-table-column>
                    </el-table>
                </div>
                <div style="width: 20%;display: flex;flex-direction: column;justify-content: space-around">
                    <el-button style="width: 150px;margin-left: 10px">导出为EXCEL</el-button>
                    <el-button style="width: 150px">导出为CSV</el-button>
                    <el-button style="width: 100px">打印</el-button>
                </div>
            </div>
        </div>
        <!--limit:最大上传数，on-exceed：超过最大上传数量时的操作  -->
    </div>
</template>

<script>
    // ,character
import { readFile } from '../../assets/lib/utils'
import { getResultTest,getSchRegion } from '../../assets/lib/getResult'
import {getMap} from '../../assets/lib/ChinaMapShow'
import {getschool} from '../../assets/lib/homeserve'
import xlsx from 'xlsx';


    export default {
        name: "",
        data(){
            return{
                limitUpload:1,//上传excel时，同时允许上传的最大数
                fileList:[],//excel文件列表
                checkboxGroup1:[],
                bcols:[],//有几行
                sch_fields:[],//可能是学校的字段
                rowbegin:0,//开始行
                rowend:0,//结束行
                schfiled:'选择学校名称字段',//用户选择的学校字段
                mapresult:[],
                otherdata:[],
                schNames:[],
                excelDatas:[],//表格内拿到的所有数据
                err_schname:true,//是否正确获取了学校列
                school_id:[],//获取全部的学校代码和名字
                name_id:[]//代码和名字对应
            }
        },
        methods: {
            handleIndexBegin(command){
                this.rowbegin = command;
            },
            handleIndexEnd(command){
                this.rowend = command;
            },
            handleSchField(command){
                this.schfiled = command;
                this.$message('click on schfiled ' + this.schfiled);
            },
            printparams(){
                this.schNames = [];
                console.log(this.mapresult.length);
                if (this.rowbegin>this.rowend || this.rowbegin===0){
                    this.$message("左侧行数应小于右侧行数");
                    return;
                }
                for (let i=this.rowbegin-1;i<this.rowend;i++){
                    this.schNames.push(this.excelDatas[i][this.schfiled]);
                }
                console.log("schnamelen = "+this.schNames.length)
                getResultTest(this.schNames).then(res=>{
                    this.mapresult = res;
                    console.log(res);
                    console.log("res = "+JSON.stringify(res));
                })
            },
            getResult(){
                this.schNames = [];
                if (this.rowbegin>this.rowend || this.rowbegin===0){
                    this.$message("左侧行数应小于右侧行数");
                    return;
                }
                for (let i=this.rowbegin-1;i<this.rowend;i++){
                    this.schNames.push(this.excelDatas[i][this.schfiled]);
                }

                console.log("schNames len = "+this.schNames.length);

                for (let i=0;i<this.schNames.length;i++){
                    getSchRegion(this.name_id[this.schNames[i]]).then(res=>{
                        var loc = res.location;
                        let i = 0;
                        for (i=0;i<this.mapresult.length;i++){
                            if (this.mapresult[i].name === loc){
                                this.mapresult[i].value +=1;
                                break;
                            }
                        }
                        if (i===this.mapresult.length){
                            this.mapresult.push({name: loc,value: 1});
                        }
                        getMap(this.mapresult);
                    }).catch(error=>{
                        console.log(error);
                        this.err_schname = false;
                    })
                }

            },
            exportEXCEL(type){
                console.log("进入了导出EXCEL函数")
                let arr = this.mapresult.map(item=>{
                    return {
                        地区:item.name,
                        人数:item.value,
                    };
                });
                let sheet = xlsx.utils.json_to_sheet(arr);
                let book = xlsx.utils.book_new();
                xlsx.utils.book_append_sheet(book,sheet,"sheet1");
                if (type === 'xlsx'){
                    xlsx.writeFile(book,`user${new Date().getTime()}.xlsx`);
                }else {
                    xlsx.writeFile(book,`user${new Date().getTime()}.csv`);
                }

            },
            async handleChange(ev){
              let file = ev.raw;
              if (!file) return;

              this.bcols = [];
              this.sch_fields = [];

              let data = await readFile(file);
              let workbook = xlsx.read(data,{type:'binary'});

              let worksheet = workbook.Sheets[workbook.SheetNames[1]];
              data = xlsx.utils.sheet_to_json(worksheet);
              console.log(data);

              const range = xlsx.utils.decode_range(worksheet['!ref']);
              let C ,R = range.s.r;
              for (C = range.s.c;C<=range.e.c;++C){
                  const cell = worksheet[xlsx.utils.encode_cell({c:C,r:R})]
                  let hdr = "UNKNOWN" + C;
                  if (cell && cell.t) hdr = xlsx.utils.format_cell(cell);
                  this.sch_fields.push({'field':hdr});
              }

              let datalen = data.length;
              for (var i = 1;i<=datalen;i++){
                  this.bcols.push({'page':i});
              }

              this.excelDatas = data;
            },
        },
        mounted() {
             getMap([]);
        },
        created() {
            getschool().then(res=>{
                this.school_id = res.data;
                for (let i=0;i<this.school_id.length;i++){
                    this.name_id[this.school_id[i].cname] = this.school_id[i].cid;
                }
            })
        }
    }

</script>

<style scoped>
    .screen {
        width: 100%;
        height: 300px;
        border: 1px solid darkgrey;
    }
    #source_data,#data_scale,#field_map,#finish,#conditions{
        display: flex;
        justify-content: start;
        margin-left: 20px;
    }

    #finish{
        width: 120px;
        padding-left: 46px;
    }
</style>
