<template>
  <div class="component-upload-image">
    <el-upload multiple :action="uploadImgUrl+url" list-type="picture-card" :on-success="handleUploadSuccess"
      :before-upload="handleBeforeUpload" :limit="limit" :on-error="handleUploadError" :on-exceed="handleExceed"
      name="file" :on-remove="handleRemove" :show-file-list="true" :headers="headers" :file-list="fileList"
      :on-preview="handlePictureCardPreview" :class="{hide: this.fileList.length >= this.limit}">
      <i class="el-icon-plus"></i>
    </el-upload>

    <!-- 上传提示 -->
    <div class="el-upload__tip" slot="tip" v-if="showTip">
      请上传
      <template v-if="fileSize"> 大小不超过 <b style="color: #f56c6c">{{ fileSize }}MB</b> </template>
      <template v-if="fileType"> 格式为 <b style="color: #f56c6c">{{ fileType.join("/") }}</b> </template>
      的文件
    </div>

    <el-dialog :visible.sync="dialogVisible" title="预览" width="800" append-to-body>
      <img :src="dialogImageUrl" style="display: block; max-width: 100%; margin: 0 auto" />
    </el-dialog>
  </div>
</template>

<script>
  import {
    getToken
  } from "@/utils/auth";
  import {
    delOss
  } from "@/api/system/oss";

  export default {
    props: {
      value: [String, Object, Array],
      // 图片数量限制
      limit: {
        type: Number,
        default: 5,
      },
      // 大小限制(MB)
      fileSize: {
        type: Number,
        default: 5,
      },
      // 文件类型, 例如['png', 'jpg', 'jpeg']
      fileType: {
        type: Array,
        default: () => ["png", "jpg", "jpeg"],
      },
      // 是否显示提示
      isShowTip: {
        type: Boolean,
        default: true
      },
      url: {
        type: String,
        default: "/goods/category/upload"
      }
    },
    data() {
      return {
        number: 0,
        uploadList: [],
        dialogImageUrl: "",
        dialogVisible: false,
        hideUpload: false,
        baseUrl: process.env.VUE_APP_BASE_API,
        uploadImgUrl: process.env.VUE_APP_BASE_API, // 上传的图片服务器地址
        headers: {
          Authorization: "Bearer " + getToken(),
        },
        fileList: [],
        id: null
      };
    },
    watch: {
      value: {
        handler(val) {
          console.log("my upload -----")
          console.log(val)
          if (val.length == 1) {
            // 首先将值转为数组
            console.log("------ val upload ------")
            console.log(val)
            if (val[0].url == '' || val[0].url == null) {
              this.id = val[0].name
              this.fileList = []
            } else {
              // 然后将数组转为对象数组
              this.fileList = val
            }

          } else {
            this.fileList = [];
            return [];
          }
        },
        deep: true,
        immediate: true
      }
    },
    computed: {
      // 是否显示提示
      showTip() {
        return this.isShowTip && (this.fileType || this.fileSize);
      },
    },
    methods: {
      // 删除图片
      handleRemove(file, fileList) {
        const findex = this.fileList.map(f => f.name).indexOf(file.name);
        if (findex > -1) {
          const cat = this.fileList[findex]
          console.log("-----findex----")
          console.log(cat)
          const update = {
            catId: this.id,
            catIcon: cat.url
          }
          this.$emit("delOss", update)
          this.fileList.splice(findex, 1);
          this.$emit("input", this.fileList);
        }
      },
      // 上传成功回调
      handleUploadSuccess(res) {
        if (res.code == 200) {
          this.fileList = []
          if (this.id == -1) {
            this.fileList.push({
              url: res.data.url,
            })
          } else {
            this.fileList.push({
              url: res.data.url,
              name: this.id
            })
          }

          this.$emit("input", this.fileList);
          this.$modal.closeLoading();

        } else {
          this.$modal.msgError(res.msg);
          this.$modal.closeLoading();
        }
      },
      // 上传前loading加载
      handleBeforeUpload(file) {
        let isImg = false;
        if (this.fileType.length) {
          let fileExtension = "";
          if (file.name.lastIndexOf(".") > -1) {
            fileExtension = file.name.slice(file.name.lastIndexOf(".") + 1);
          }
          isImg = this.fileType.some((type) => {
            if (file.type.indexOf(type) > -1) return true;
            if (fileExtension && fileExtension.indexOf(type) > -1) return true;
            return false;
          });
        } else {
          isImg = file.type.indexOf("image") > -1;
        }

        if (!isImg) {
          this.$modal.msgError(`文件格式不正确, 请上传${this.fileType.join("/")}图片格式文件!`);
          return false;
        }
        if (this.fileSize) {
          const isLt = file.size / 1024 / 1024 < this.fileSize;
          if (!isLt) {
            this.$modal.msgError(`上传头像图片大小不能超过 ${this.fileSize} MB!`);
            return false;
          }
        }
        this.$modal.loading("正在上传图片，请稍候...");
      },
      // 文件个数超出
      handleExceed() {
        this.$modal.msgError(`上传文件数量不能超过 ${this.limit} 个!`);
      },
      // 上传失败
      handleUploadError(res) {
        this.$modal.msgError("上传图片失败，请重试");
        this.$modal.closeLoading();
      },
      // 预览
      handlePictureCardPreview(file) {
        this.dialogImageUrl = file.url;
        this.dialogVisible = true;
      },
    }
  };
</script>
<style scoped lang="scss">
  // .el-upload--picture-card 控制加号部分
  ::v-deep.hide .el-upload--picture-card {
    display: none;
  }

  // 去掉动画效果
  ::v-deep .el-list-enter-active,
  ::v-deep .el-list-leave-active {
    transition: all 0s;
  }

  ::v-deep .el-list-enter,
  .el-list-leave-active {
    opacity: 0;
    transform: translateY(0);
  }
</style>
