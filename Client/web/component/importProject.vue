<template>
    <el-dialog title="导入"  size="small" ref="box" v-model="showDialog">
        <el-row class="row" style="height: 50px;line-height: 50px">
            导入类型：&nbsp;&nbsp;&nbsp;
            <el-select v-model="type">
                <el-option label="PostMan V2 JSON" :value="0"></el-option>
                <el-option label="DOClever JSON" :value="1"></el-option>
                <el-option label="RAP JSON" :value="2"></el-option>
                <el-option label="Swagger" :value="3"></el-option>
            </el-select>
        </el-row>
        <el-row class="row" style="height: 1px;background-color: rgb(247,246,242);margin-bottom: 10px"></el-row>
        <el-row class="row" v-show="type==0">
            <el-input v-drag="'text'" type="textarea" :rows="10" placeholder="请输入JSON" v-model="text" style="margin-bottom: 10px">
            </el-input>
            请编辑BaseUrl：
            <el-checkbox v-model="ignore" :true-label="1" :false-label="0" style="float: right;margin-right: 20px">
                忽略大小写
            </el-checkbox>
            <template v-for="(item,index) in arr">
                <el-row class="row" style="height: 50px;line-height: 50px;text-align: center">
                    <el-col class="col" :span="20">
                        <el-input style="width: 100%" placeholder="请填写baseurl地址" v-model="item.title"></el-input>
                    </el-col>
                    <el-col class="col" :span="2">
                        <el-button type="text" size="small" style="color: red;font-size: 15px;" @click="remove(index)" icon="close"></el-button>
                    </el-col>
                    <el-col class="col" :span="2">
                        <el-button style="font-size: 15px" v-if="index==arr.length-1" @click="arr.push({title:''})" size="small" type="text" icon="plus"></el-button>
                    </el-col>
                </el-row>
            </template>
        </el-row>
        <el-row class="row" v-show="type==1">
            <el-input v-drag="'textMy'" type="textarea" :rows="10" placeholder="请输入JSON" v-model="textMy">
            </el-input>
        </el-row>
        <el-row class="row" v-show="type==2">
            <el-input v-drag="'textRap'" type="textarea" :rows="10" placeholder="请输入JSON" v-model="textRap">
            </el-input>
            <el-row class="row" style="height:20px">
            </el-row>
            <el-row class="row">
                Body Type:&nbsp;&nbsp;&nbsp;
                <el-select v-model="rapBodyType">
                    <el-option :value="0" label="x-www-form-urlencoded"></el-option>
                    <el-option :value="1" label="application/json"></el-option>
                </el-select>
            </el-row>
        </el-row>
        <el-row class="row" v-show="type==3">
            Swagger类型:&nbsp;&nbsp;&nbsp;
            <el-select v-model="swaggerType">
                <el-option :value="0" label="URL"></el-option>
                <el-option :value="1" label="JSON"></el-option>
            </el-select><br><br>
            <el-input placeholder="请输入Swagger URL" v-model="textSwaggerURL" v-show="swaggerType==0"></el-input>
            <el-input v-drag="'textSwaggerJSON'" type="textarea" :rows="10" placeholder="请输入Swagger Url" v-model="textSwaggerJSON" v-show="swaggerType==1"></el-input>
        </el-row>
        <el-row class="row">{{status}}</el-row>
        <el-row class="dialog-footer" slot="footer">
            <el-button type="primary" :loading="savePending" @click="save">
                保存
            </el-button>
        </el-row>
    </el-dialog>
</template>

<script>
    var uuid=require("uuid");
    var dragFile=require("../director/dragFile");
    var bus=require("../bus/bus");
    module.exports={
        data:function () {
            return {
                type:0,
                text:"",
                textMy:"",
                arr:[{
                    title:""
                }],
                savePending:false,
                status:"",
                ignore:0,
                textRap:"",
                rapBodyType:0,
                swaggerType:0,
                textSwaggerJSON:"",
                textSwaggerURL:"",
                showDialog:false
            }
        },
        directives:{
            drag:dragFile
        },
        computed:{

        },
        methods:{
            remove:function (index) {
                if(this.arr.length>1)
                {
                    this.arr.splice(index,1)
                }
                else
                {
                    this.arr[0].title="";
                }
            },
            save:function () {
                var _this=this;
                function postman(obj,arr)
                {
                    if(!obj.info.name)
                    {
                        $.tip("项目名称为空",0);
                        return;
                    }
                    _this.savePending=true;
                    _this.status="正在创建项目"+obj.info.name;
                    var projectID,groupID;
                    var update={
                        name:obj.info.name,
                        dis:obj.info.description,
                        import:1
                    };
                    if(session.get("teamId"))
                    {
                        update.team=session.get("teamId")
                    }
                    var insertDate;
                    var pro=net.post("/project/create",update).then(function (data) {
                        if(data.code==200)
                        {
                            _this.$store.commit("addProjectCreate",data.data);
                            insertDate=data.data;
                            projectID=data.data._id;
                        }
                        else
                        {
                            throw data.msg;
                        }
                    });
                    var count=0,indexInterface=0,bDefaultGroup=false,bDefaultGroupId;
                    obj.item.forEach(function (o) {
                        if(o.item)
                        {
                            count+=o.item.length;
                        }
                        else
                        {
                            count++;
                        }
                    })
                    obj.item.forEach(function (group) {
                        pro=pro.then(function () {
                            var groupName,bDefault=false;
                            if(group.item)
                            {
                                groupName=group.name;
                            }
                            else
                            {
                                bDefault=true;
                                if(!bDefaultGroup)
                                {
                                    groupName="未命名";
                                    bDefaultGroup=true;
                                }
                                else
                                {
                                    groupID=bDefaultGroupId;
                                    return;
                                }
                            }
                            _this.status="正在创建分组"+groupName;
                            var query={};
                            query.id=projectID;
                            query.name=groupName;
                            query.import=1;
                            return net.post("/group/create",query).then(function (data) {
                                if(data.code!=200)
                                {
                                    throw data.msg;
                                }
                                else
                                {
                                    groupID=data.data._id;
                                    if(bDefault)
                                    {
                                        bDefaultGroupId=groupID;
                                    }
                                }
                            })
                        })
                        if(!group.item)
                        {
                            group.item=[group];
                        }
                        group.item.forEach(function (item) {
                            pro=pro.then(function () {
                                indexInterface++;
                                insertDate.interfaceCount=indexInterface;
                                _this.status="正在导入第"+indexInterface+"个接口"+item.name+"，一共"+count+"个接口";
                                var obj;
                                if(typeof(item.request.url)=="object")
                                {
                                    var objUrl=$.parseURL(item.request.url.raw);
                                    var url=objUrl.source;
                                    var index=url.indexOf("?");
                                    if(index>-1)
                                    {
                                        url=url.substr(0,index);
                                    }
                                    for(var i=0;i<arr.length;i++)
                                    {
                                        if(_this.ignore)
                                        {
                                            index=url.toLowerCase().indexOf(arr[i].toLowerCase());
                                        }
                                        else
                                        {
                                            index=url.indexOf(arr[i]);
                                        }
                                        if(index>-1)
                                        {
                                            url=url.substr(index+arr[i].length);
                                            break;
                                        }
                                    }
                                    if(item.request.url.path)
                                    {
                                        item.request.url.path.forEach(function (obj) {
                                            if(obj[0]==":")
                                            {
                                                url=url.replace(obj,"{"+obj.substr(1)+"}");
                                            }
                                        })
                                    }
                                    obj={
                                        name:item.name,
                                        url:url,
                                        group:groupID,
                                        remark:item.request.description,
                                        project:projectID,
                                        method:item.request.method,
                                        finish:1,
                                        param:[{
                                            before:{
                                                mode:0,
                                                code:""
                                            },
                                            after:{
                                                mode:0,
                                                code:""
                                            },
                                            name:"未命名",
                                            remark:"",
                                            id:uuid()
                                        }]
                                    };
                                    var restParam=[];
                                    if(item.request.url.variable)
                                    {
                                        item.request.url.variable.forEach(function (obj) {
                                            restParam.push({
                                                name:obj.key,
                                                remark:obj.description,
                                                value:[obj.value]
                                            })
                                        })
                                    }
                                    obj.param[0].restParam=restParam;
                                    var param=[];
                                    for(var key in objUrl.params)
                                    {
                                        var v={
                                            name:key,
                                            must:1,
                                            remark:""
                                        };
                                        if(objUrl.params[key]!=="" && objUrl.params[key]!==undefined)
                                        {
                                            v.value=[objUrl.params[key]];
                                        }
                                        param.push(v);
                                    }
                                    obj.param[0].queryParam=param;
                                }
                                else
                                {
                                    var objUrl=$.parseURL(item.request.url);
                                    var url=objUrl.source,index=url.indexOf("?");
                                    if(index>-1)
                                    {
                                        url=url.substr(0,index);
                                    }
                                    for(var i=0;i<arr.length;i++)
                                    {
                                        if(_this.ignore)
                                        {
                                            index=url.toLowerCase().indexOf(arr[i].toLowerCase());
                                        }
                                        else
                                        {
                                            index=url.indexOf(arr[i]);
                                        }
                                        if(index>-1)
                                        {
                                            url=url.substr(index+arr[i].length);
                                            break;
                                        }
                                    }
                                    obj={
                                        name:item.name,
                                        url:url,
                                        group:groupID,
                                        remark:item.request.description,
                                        project:projectID,
                                        method:item.request.method,
                                        finish:1,
                                        param:[{
                                            before:{
                                                mode:0,
                                                code:""
                                            },
                                            after:{
                                                mode:0,
                                                code:""
                                            },
                                            name:"未命名",
                                            remark:"",
                                            id:uuid()
                                        }]
                                    };
                                    var param=[];
                                    for(var key in objUrl.params)
                                    {
                                        var v={
                                            name:key,
                                            must:1,
                                            remark:""
                                        };
                                        if(objUrl.params[key]!=="" && objUrl.params[key]!==undefined)
                                        {
                                            v.value=[objUrl.params[key]];
                                        }
                                        param.push(v);
                                    }
                                    obj.param[0].queryParam=param;
                                    obj.param[0].restParam=[];
                                }
                                var bJSON=false;
                                obj.param[0].header=item.request.header.map(function (obj) {
                                    if(obj.key.toLowerCase()=="content-type" && obj.value.toLowerCase()=="application/json")
                                    {
                                        bJSON=true;
                                    }
                                    return {
                                        name:obj.key,
                                        value:obj.value,
                                        remark:""
                                    }
                                })
                                if(obj.method.toLowerCase()=="post" || obj.method.toLowerCase()=="put" || obj.method.toLowerCase()=="patch")
                                {
                                    var body,bodyInfo;
                                    if(item.request.body.mode=="urlencoded" || item.request.body.mode=="formdata")
                                    {
                                        bodyInfo={
                                            type:0,
                                            rawType:0,
                                            rawTextRemark:"",
                                            rawFileRemark:"",
                                            rawText:"",
                                        };
                                        body=item.request.body[item.request.body.mode].map(function (obj)
                                        {
                                            var o={
                                                name:obj.key,
                                                type:obj.type=="text"?0:1,
                                                must:1,
                                                remark:"",
                                            }
                                            if(o.type==0 && obj.value!=="" && obj.value!==undefined)
                                            {
                                                o.value=[obj.value];
                                            }
                                            return o;
                                        })
                                    }
                                    else if(item.request.body.mode=="raw")
                                    {
                                        body=[];
                                        if(bJSON)
                                        {
                                            var objJSON;
                                            try
                                            {
                                                objJSON=eval("("+item.request.body.raw+")");
                                            }
                                            catch (err)
                                            {

                                            }
                                            if(objJSON)
                                            {
                                                var result=[];
                                                for(var key in objJSON)
                                                {
                                                    helper.handleResultData(key,objJSON[key],result,null,1)
                                                }
                                                bodyInfo={
                                                    type:1,
                                                    rawType:2,
                                                    rawTextRemark:"",
                                                    rawFileRemark:"",
                                                    rawText:"",
                                                    rawJSON:result
                                                };
                                            }
                                            else
                                            {
                                                bodyInfo={
                                                    type:1,
                                                    rawType:0,
                                                    rawTextRemark:"",
                                                    rawFileRemark:"",
                                                    rawText:item.request.body.raw,
                                                };
                                            }
                                        }
                                        else
                                        {
                                            bodyInfo={
                                                type:1,
                                                rawType:0,
                                                rawTextRemark:"",
                                                rawFileRemark:"",
                                                rawText:item.request.body.raw,
                                            };
                                        }
                                    }
                                    else
                                    {
                                        body=[];
                                        bodyInfo={
                                            type:0,
                                            rawType:0,
                                            rawTextRemark:"",
                                            rawFileRemark:"",
                                            rawText:"",
                                        };
                                    }
                                    obj.param[0].bodyParam=body;
                                    obj.param[0].bodyInfo=bodyInfo
                                }
                                obj.param[0].outParam=[];
                                obj.param[0].outInfo={
                                    type:0,
                                    rawRemark:"",
                                    rawMock:"",
                                };
                                obj.param=JSON.stringify(obj.param);
                                return net.post("/interface/create",obj).then(function (data) {
                                    if(data.code!=200)
                                    {
                                        throw data.msg
                                    }
                                });
                            })
                        })
                        if(group.item && group.item[0]==group)
                        {
                            group.item=null;
                        }
                    })
                    pro=pro.then(function () {
                        return net.put("/project/url",{
                            id:projectID,
                            urls:JSON.stringify(arr.map(function (obj) {
                                return {
                                    url:obj,
                                    remark:""
                                }
                            }))
                        }).then(function (data) {
                            if(data.code!=200)
                            {
                                throw data.msg
                            }
                        })
                    }).then(function () {
                        _this.savePending=false;
                        _this.$refs.box.close();
                        $.notify("导入成功",1);
                        if(session.get("teamId"))
                        {
                            bus.$emit("updateTeamProject",1,count);
                        }
                    }).catch(function (err) {
                        _this.savePending=false;
                        $.tip(err,0);
                    })
                }
                if(this.type==0)
                {
                    if(!this.text)
                    {
                        $.tip("请输入JSON",0);
                        return;
                    }
                    var obj;
                    try
                    {
                        obj=JSON.parse(this.text)
                    }
                    catch(e)
                    {
                        $.tip("JSON格式有错误",0);
                        return;
                    }
                    if(!obj.info._postman_id)
                    {
                        $.tip("不是可识别的JSON格式",0);
                        return;
                    }
                    var arr=[];
                    this.arr.forEach(function (obj) {
                        if(obj.title)
                        {
                            arr.push(obj.title);
                        }
                    })
                    if(arr.length==0)
                    {
                        $.tip("请输入BaseUrl",0);
                        return;
                    }
                    postman(obj,arr)
                }
                else if(this.type==1)
                {
                    if(!this.textMy)
                    {
                        $.tip("请输入JSON",0);
                        return;
                    }
                    var obj;
                    try
                    {
                        obj=JSON.parse(this.textMy);
                    }
                    catch (err)
                    {
                        $.tip("json解析错误",0);
                        return;
                    }
                    if(obj.flag!="SBDoc")
                    {
                        $.tip("不是DOClever的导出格式",0);
                        return;
                    }
                    var _this=this;
                    this.savePending=true;
                    var update={
                        json:this.textMy
                    };
                    if(session.get("teamId"))
                    {
                        update.team=session.get("teamId")
                    }
                    net.post("/project/importjson",update).then(function (data) {
                        _this.savePending=false;
                        if(data.code==200)
                        {
                            $.notify("导入成功",1);
                            if(session.get("teamId"))
                            {
                                _this.$parent.obj.project.unshift(data.data);
                                bus.$emit("updateTeamProject",1,data.data.interfaceCount);
                            }
                            else
                            {
                                _this.$store.commit("addProjectCreate",data.data);
                            }
                            _this.$refs.box.close();
                        }
                        else
                        {
                            $.notify(data.msg,0);
                        }
                    })
                }
                else if(this.type==2)
                {
                    if(!this.textRap)
                    {
                        $.tip("请输入JSON",0);
                        return;
                    }
                    var obj;
                    try
                    {
                        obj=JSON.parse(this.textRap);
                        obj=eval("("+obj.modelJSON+")");
                    }
                    catch(e)
                    {
                        $.tip("JSON格式有错误",0);
                        return;
                    }
                    var _this=this;
                    this.savePending=true;
                    var update={
                        json:JSON.stringify(obj),
                        bodytype:this.rapBodyType
                    };
                    if(session.get("teamId"))
                    {
                        update.team=session.get("teamId")
                    }
                    net.post("/project/importrap",update).then(function (data) {
                        _this.savePending=false;
                        if(data.code==200)
                        {
                            $.notify("导入成功",1);
                            if(session.get("teamId"))
                            {
                                _this.$parent.obj.project.unshift(data.data);
                                bus.$emit("updateTeamProject",1,data.data.interfaceCount);
                            }
                            else
                            {
                                _this.$store.commit("addProjectCreate",data.data);
                            }
                            _this.$refs.box.close();
                        }
                        else
                        {
                            $.notify(data.msg,0);
                        }
                    })
                }
                else if(this.type==3)
                {
                    if(this.swaggerType==0)
                    {
                        if(!this.textSwaggerURL)
                        {
                            $.tip("请输入url地址",0);
                            return;
                        }
                    }
                    else if(this.swaggerType==1)
                    {
                        if(!this.textSwaggerJSON)
                        {
                            $.tip("请输入JSON",0);
                            return;
                        }
                    }
                    var obj;
                    if(this.swaggerType==1)
                    {
                        try
                        {
                            obj=JSON.parse(this.textSwaggerJSON);
                        }
                        catch(e)
                        {
                            $.tip("JSON格式有错误",0);
                            return;
                        }
                    }
                    var _this=this;
                    this.savePending=true;
                    var update={};
                    if(this.swaggerType==0)
                    {
                        update.url=this.textSwaggerURL;
                    }
                    else
                    {
                        update.json=this.textSwaggerJSON;
                    }
                    if(session.get("teamId"))
                    {
                        update.team=session.get("teamId")
                    }
                    net.post("/project/importswagger",update).then(function (data) {
                        _this.savePending=false;
                        if(data.code==200)
                        {
                            $.notify("导入成功",1);
                            if(session.get("teamId"))
                            {
                                _this.$parent.obj.project.unshift(data.data);
                                bus.$emit("updateTeamProject",1,data.data.interfaceCount);
                            }
                            else
                            {
                                _this.$store.commit("addProjectCreate",data.data);
                            }
                            _this.$refs.box.close();
                        }
                        else
                        {
                            $.notify(data.msg,0);
                        }
                    })
                }
            }
        }
    }
</script>
