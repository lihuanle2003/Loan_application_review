<template>
  <div>
    <el-empty
      v-if="InstDataComputed.length === 0 ? true : false"
      description="暂无数据"
    ></el-empty>
    <el-card
      v-if="dataToShow.length > 0 ? true : false"
      style="
        margin-bottom: 20px;
        border-bottom: 3px solid rgba(100, 100, 100.8);
      "
    >
      <div class="userInfo">
        <span class="el-icon-s-custom">
          <!-- 还需要展示更多数据 -->
          姓名
        </span>
        <span class="el-icon-phone"> 电话号码 </span>
        <span class="el-icon-male"> 性别 </span>
      </div>
      <div class="useraddr">
        <span class="el-icon-map-location"> 地址 </span>
        <span> 身份证号 </span>
      </div>
      <div class="button">
        <span>操 作</span>
      </div>
    </el-card>

    <el-row
      v-for="(item, index) in dataToShow"
      :key="index"
      style="margin-bottom: 10px"
      v-loading="loadstatus"
    >
      <el-col>
        <el-card>
          <div class="userInfo">
            <span>
              <!-- 还需要展示更多数据 -->
              {{ item.name }}
            </span>
            <span>
              {{ item.phoneNum }}
            </span>
            <span>
              {{ item.sex }}
            </span>
          </div>
          <div class="useraddr">
            <span>
              {{ item.addr }}
            </span>
            <span>
              {{ item.idCardNum }}
            </span>
          </div>
          <div class="button">
            <!-- 通过按钮 -->
            <el-button
              type="primary"
              @click="success(item)"
              icon="el-icon-check"
              plain
              >通过</el-button
            >
            <!-- 驳回按钮 -->
            <el-button
              v-if="btnShow"
              type="danger"
              @click="(dialogFormVisible = true), (currentData = item)"
              icon="el-icon-circle-close"
              plain
              >驳回</el-button
            >
          </div>
        </el-card>
      </el-col>
    </el-row>

    <!-- 驳回弹窗 填写备注 操作员 -->
    <el-dialog
      title="驳回确认"
      width="30%"
      :visible.sync="dialogFormVisible"
      center
      :before-close="handleColse"
    >
      <!-- 主体内容 -->
      <el-form
        ref="form"
        :model="dia_form"
        :label-position="formLabelPosition"
        :rules="diaForm_rules"
      >
        <el-form-item label="审核人员">
          <el-input v-model="dia_form.applic_name" :disabled="true"></el-input>
        </el-form-item>
        <el-form-item label="备注信息" prop="remark">
          <el-input v-model="dia_form.remark" type="textarea"></el-input>
        </el-form-item>
      </el-form>
      <!-- 弹窗底部按钮 -->
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogFormVisible = false">取 消</el-button>
        <el-button type="primary" @click="vaildDialogForm()">确 定</el-button>
      </span>
    </el-dialog>

    <el-pagination
      background
      layout="prev, pager, next"
      :page-size="page_size"
      :total="InstDataComputed.length"
      :current-page="currentPage"
      @current-change="handleChangePage"
    >
    </el-pagination>
  </div>
</template>

<script>
import { firstInstSuccess, firstInstNoArgee } from "@/apis/submitInst";
import jsCookie from "js-cookie";
export default {
  name: "InstUserTable",
  props: ["InstData", "btnShow", "title", "num"],
  data() {
    return {
      page_size: 6,
      currentPage: 0,

      // 展示数据
      dataToShow: [],

      // 加载状态
      loadstatus: true,
      dialogFormVisible: false,
      currentData: "",
      dia_form: {
        applic_name: jsCookie.get("userName"),
        remark: "",
      },
      formLabelPosition: "top",
      // 弹窗验证规则
      diaForm_rules: {
        remark: [
          { required: true, message: "请填写驳回备注", targger: "blur" },
        ],
      },
    };
  },

  components: {},

  // 使用计算属性 缓存页面数据
  computed: {
    InstDataComputed(){
      return this.InstData
    }
  },
  watch: {
    dataToShow:{
      deep:true,
      handler(newVal,oldVal){
        // 非第一页展示数据操作完 向前跳跃一个页面
        if(!newVal.length && this.currentPage !=0){
          this.handleChangePage(this.currentPage - 1)
        }
      }
    }
  },

  created() {},

  mounted() {
    // 放如任务队列 靠后执行 等待vuex ajax微任务执行完成后执行
    setTimeout(() => {
      this.getData();
    }, 0);
  },

  methods: {
    getData() {
      this.sliceDataToShow(0, this.page_size);
    },

    /**
     * 弹窗关闭前处理业务
     */
    handleColse() {
      // 清空表单内容
      this.$refs.form.resetFields();
    },
    /**
     * 弹窗表单内容验证函数
     */
    vaildDialogForm() {
      this.$refs.form.validate((valid) => {
        if (valid) {
          this.noArgee(this.currentData);
          this.dialogFormVisible = false;
          this.handleColse();
        } else {
          this.$message.warning("请完善表单内容");
        }
      });
    },

    /**
     * 驳回操作
     */
    noArgee(item) {
      // 组装请求对象
      const requestObj = {
        id: item.id,
        idCardNum: item.idCardNum,
        message: this.dia_form.remark,
        sex: item.sex,
        status: 4,
      };
      this.$confirm(`是否驳回${item.name}的${this.title}请求`, "驳回请求", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(async () => {
          // 调用接口修数据状态为驳回
          let res = await firstInstNoArgee(requestObj);
          if (res.data.code === 200) {
            this.resPageBy$Delete(item.id)
            this.$message({
              type: "success",
              message: res.data.message,
            });
          } else if (res.data.code === 605) {
            this.$message({
              type: "error",
              message: "服务器错误",
            });
          }
        })
        .catch(() => {
          this.$message({
            type: "info",
            message: "取消",
          });
        });
    },

    /**
     * 同意操作
     */
    success(item) {
      this.$confirm(`是否通过${item.name}的${this.title}请求`, "通过请求", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning",
      })
        .then(async () => {
          let res = await firstInstSuccess({
            id: item.id,
            status: this.num,
            idCardNum: item.idCardNum,
            sex: item.sex,
          });
          if (res.data.code === 200) {
            this.resPageBy$Delete(item.id);
            this.$message({
              type: "success",
              message: "审核通过",
            });
          } else if (res.data.code === 605) {
            this.$message({
              type: "error",
              message: "服务器错误",
            });
          }
        })
        .catch((e) => {
          console.log(e);
          this.$message({
            type: "info",
            message: "取消",
          });
        });
    },

    // 分页函数
    handleChangePage(page) {
      this.currentPage = page;
      if (page > 0) {
        let str = -this.page_size + this.page_size * page;
        let end = 0 + this.page_size * page;
        this.sliceDataToShow(str, end);
      } else {
        this.sliceDataToShow(0, this.page_size);
      }
    },
    // 切割函数
    sliceDataToShow(str, end) {
      this.dataToShow = this.InstData.slice(str, end);
      this.loadstatus = false;
    },

    /**
     * 通过 或者 驳回操作后 使用$delete修改数组数据 响应到页面
     */
    resPageBy$Delete(id) {
      let index = this.InstData.findIndex((item) => item.id === id);
      if (index != -1) this.$delete(this.InstData, index);
      // 删除数据后，重新根据currentPage划分页面
      this.handleChangePage(this.currentPage);
    },
  },
};
</script>
<style lang="less">
.el-card__body {
  padding: 0;
  width: 100%;
  display: flex;
  justify-content: space-between;
}
.userInfo {
  background-color: rgba(118, 129, 153, 0.8);
}
.useraddr {
  background-color: rgba(181, 200, 209, 0.8);
}

.userInfo,
.useraddr {
  flex: 1;
  display: flex;
  flex-wrap: wrap;
  min-width: 250px;
  justify-content: space-between;
  align-items: center;
  padding: 15px;
  border-right: 1px solid gray;
}

.button {
  display: flex;
  width: 20%;
  min-width: 140px;
  justify-content: space-around;
  align-items: center;
  padding: 15px;
  background-color: rgba(240, 253, 255, 0.8);
  .el-button {
    padding: 10px 5px;
    font-size: 1rem;
  }
}

.el-form-item {
  width: 100%;
}
</style>