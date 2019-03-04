<template>
    <section>
        <!--工具条-->
        <el-col :span="24" class="toolbar" style="padding-bottom: 0px;">
            <el-form :inline="true" :model="filters">
                <el-form-item>
                    <el-input v-model="filters.keyword" placeholder="关键字"></el-input>
                </el-form-item>
                <el-form-item>
                    <el-button type="primary" v-on:click="getProducts">查询</el-button>
                </el-form-item>
                <el-form-item>
                    <el-button type="primary" @click="handleAdd">新增</el-button>
                </el-form-item>
            </el-form>
        </el-col>

        <!--列表-->
        <el-table :data="products" highlight-current-row v-loading="listLoading" @selection-change="selsChange"
                  style="width: 100%;">
            <el-table-column type="selection" width="55">
            </el-table-column>
            <el-table-column type="index" width="60">
            </el-table-column>
            <el-table-column prop="name" label="标题" :show-overflow-tooltip="true" width="200" sortable>
            </el-table-column>
            <el-table-column prop="subName" label="副标题" :show-overflow-tooltip="true" width="200" sortable>
            </el-table-column>
            <el-table-column prop="onSaleTimeStr" label="上架时间" width="200" sortable>
            </el-table-column>
            <el-table-column prop="offSaleTimeStr" label="下架时间" min-width="200" sortable>
            </el-table-column>
            <el-table-column prop="state" label="状态" min-width="120" :formatter="formatState" sortable>
            </el-table-column>
            <el-table-column label="操作" width="200">
                <template scope="scope">
                    <el-button size="small" @click="handleEdit(scope.$index, scope.row)">编辑</el-button>
                    <el-button type="danger" size="small" @click="handleDel(scope.$index, scope.row)">删除</el-button>
                </template>
            </el-table-column>
        </el-table>

        <!--工具条-->
        <el-col :span="24" class="toolbar">
            <el-button type="danger" @click="batchRemove" :disabled="this.sels.length===0">批量删除</el-button>
            <el-pagination layout="prev, pager, next" @current-change="handleCurrentChange" :page-size="10"
                           :total="total" style="float:right;">
            </el-pagination>
        </el-col>
        <!--编辑界面-->
        <el-dialog title="编辑" v-model="formVisible" :close-on-click-modal="false">
            <el-form :model="form" label-width="80px" :rules="formRules" ref="form">
                <el-form-item label="标题" prop="name">
                    <el-input v-model="form.name" auto-complete="off"></el-input>
                </el-form-item>
                <el-form-item label="副标题" prop="subName">
                    <el-input v-model="form.subName" auto-complete="off"></el-input>
                </el-form-item>
                <el-form-item label="品牌" prop="brandId">
                    <el-select v-model="form.brandId" clearable placeholder="请选择">
                        <el-option
                                v-for="item in brands"
                                :key="item.id"
                                :label="item.name"
                                :value="item.id">
                        </el-option>
                    </el-select>
                </el-form-item>

                <el-form-item label="商品分类" prop="productTypeId">
                    <select-tree width="200" :options="productTypes" v-model="form.productTypeId"/>
                </el-form-item>

                <el-form-item label="简介" prop="productExt.description">
                    <el-input v-model="form.productExt.description" auto-complete="off"></el-input>
                </el-form-item>

                <el-form-item label="详情" prop="productExt.richContent">
                    <div class="edit_container">
                        <quill-editor v-model="form.productExt.richContent" ref="myQuillEditor" class="editer"
                                      :options="editorOption" @ready="onEditorReady($event)"></quill-editor>
                    </div>
                </el-form-item>
            </el-form>
            <div slot="footer" class="dialog-footer">
                <el-button @click.native="formVisible = false">取消</el-button>
                <el-button type="primary" @click.native="editSubmit" :loading="editLoading">提交</el-button>
            </div>
        </el-dialog>
    </section>
</template>

<script>
    import SelectTree from '@/components/SelectTree.vue';
    import {quillEditor} from "vue-quill-editor"; //调用编辑器
    import "quill/dist/quill.core.css"
    import "quill/dist/quill.snow.css"
    import "quill/dist/quill.bubble.css"

    export default {
        computed: {
            editor() {
                return this.$refs.myQuillEditor.quill
            }
        },
        components: {
            SelectTree, quillEditor
        },
        data() { //数据
            return {
                editorOption: {},
                filters: {
                    keyword: ''
                },
                products: [],
                total: 0,
                page: 1,
                listLoading: false,
                sels: [],//列表选中列
                fileList2: [],
                formVisible: false,//编辑界面是否显示
                editLoading: false,
                formRules: {
                    name: [
                        {required: true, message: '请输入标题', trigger: 'blur'}
                    ]
                },
                staticIp: "http://172.16.7.133",
                //编辑界面数据
                form: {
                    id: 0,
                    name: '',
                    subName: '',
                    brandId: '',
                    productTypeId: '',
                    productExt: {"description": "", "richContent": ""}
                },
                defaultProps: {
                    parent: 'pid',   // 父级唯一标识
                    value: 'id',          // 唯一标识
                    label: 'name',       // 标签显示
                    children: 'children', // 子级
                },
                // 数据列表
                productTypes: []
            }
        },
        methods: { //方法\
            onEditorReady(editor) {
            },
            formatState: function (row, column) {
                return row.state === 1 ? '上架' : '下架'
            },
            handleSuccess(response, file, fileList) {
                //上传成功回调
                this.form.logo = response.object;
            },
            //删除图片
            handleRemove(response, file, fileList) {
                var filePath = this.fileList2[0].url.substr(20);
                this.$http.delete("/common/common/delete?filePath=" + filePath)
                    .then(res => {
                        if (res.data.success) {
                            this.form.logo = "";
                            this.$message({
                                message: '删除成功!',
                                type: 'success'
                            });
                        } else {
                            this.$message({
                                message: '删除失败!',
                                type: 'error'
                            });
                        }
                    })
            },
            handlePreview(file) {
                console.log(file);
            },
            handleCurrentChange(val) {
                this.page = val;
                this.getProducts();
            },
            //获取品牌的列表:
            getProducts() {
                //查询条件
                let para = {
                    page: this.page,
                    keyword: this.filters.keyword
                };
                //加载
                this.listLoading = true;
                //异步请求:
                this.$http.post("/product/product/json", para)
                    .then((res) => {
                        console.log(this);
                        this.total = res.data.total;
                        this.products = res.data.rows;
                        this.listLoading = false;
                    });
            },
            //获取品牌列表
            getBrands() {
                //加载
                this.listLoading = true;
                //异步请求:
                this.$http.get("/product/brand/list")
                    .then((res) => {
                        this.brands = res.data;
                        this.listLoading = false;
                    });
            },
            //商品分类下拉树
            getProductTypes() {
                //加载
                this.listLoading = true;
                //异步请求:
                this.$http.get("/product/productType/treeData")
                    .then(res => {
                        this.listLoading = false;
                        this.productTypes = res.data;
                    });
            },
            //删除
            handleDel: function (index, row) {
                this.$confirm('确认删除该记录吗?', '提示', {
                    type: 'warning'
                }).then(() => {
                    this.listLoading = true;
                    //NProgress.start();
                    let id = row.id;
                    this.$http.delete("/product/product/" + id)
                        .then((res) => {
                            let {msg, success, object} = res.data;
                            if (!success) {
                                this.$message({
                                    message: msg,
                                    type: 'error'
                                });
                            } else {
                                this.listLoading = false;
                                //NProgress.done();
                                this.$message({
                                    message: '删除成功',
                                    type: 'success'
                                });
                                this.getProducts();
                            }
                        });
                }).catch(() => {
                });
            },
            //显示编辑界面
            handleEdit: function (index, row) {
                this.formVisible = true;
                //回显 要提交后台
                this.form = Object.assign({}, row);
                this.fileList2.push({
                    "url": this.staticIp + row.logo,
                    "lg": row.logo
                })
            },
            //显示新增界面
            handleAdd: function () {
                this.formVisible = true;
                this.form = {
                    name: '',
                    subName: '',
                    beandId: '',
                    productTypeId: '',
                    productExt: {"description": "", "richContent": ""}
                };
            },
            //编辑
            editSubmit: function () {
                this.$refs.form.validate((valid) => {
                    if (valid) {
                        this.$confirm('确认提交吗？', '提示', {}).then(() => {
                            this.editLoading = true;
                            let para = Object.assign({}, this.form);
                            this.$http.post("/product/product/save", para).then((res) => {
                                this.editLoading = false;
                                this.$message({
                                    message: '提交成功',
                                    type: 'success'
                                });
                                this.$refs['form'].resetFields();
                                this.formVisible = false;
                                //列表刷新
                                this.getProducts();
                            });
                        });
                    }
                });
            },
            selsChange: function (sels) {
                this.sels = sels;
            },
            //批量删除
            batchRemove: function () {
                var ids = this.sels.map(item => item.id).toString();
                this.$confirm('确认删除选中记录吗？', '提示', {
                    type: 'warning'
                }).then(() => {
                    this.listLoading = true;
                    //NProgress.start();
                    let para = {ids: ids};
                    batchRemoveUser(para).then((res) => {
                        this.listLoading = false;
                        //NProgress.done();
                        this.$message({
                            message: '删除成功',
                            type: 'success'
                        });
                        this.getProducts();
                    });
                }).catch(() => {

                });
            }
        }, // $(function()) 加载完毕后执行
        mounted() {
            this.getProducts();
            this.getBrands();
            this.getProductTypes();
        }
    }

</script>

<style scoped>

</style>