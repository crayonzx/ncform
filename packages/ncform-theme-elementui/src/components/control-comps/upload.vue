<template>
  <el-upload
    ref="upload"
    :class="['ncform-upload', readonly ? 'is-read-only' : '']"
    :disabled="readonly || disabled"
    :style="{display: hidden ? 'none' : ''}"
    :action="uploadUrl"
    :multiple="mergeConfig.multiple"
    :data="mergeConfig.data"
    :show-file-list="showFileList"
    :drag="mergeConfig.drag"
    :accept="mergeConfig.accept"
    :list-type="listType"
    :auto-upload="mergeConfig.autoUpload"
    :limit="mergeConfig.limit"
    :on-change="handleUploadChange"
    :on-success="handleUploadSucess"
    :on-error="handleUploadError"
    :on-exceed="handleUploadExceed"
    :on-remove="handleUploadRemove"
    :file-list="fileList"
    :name="mergeConfig.fileField"
  >
    <!-- 可拖拽 -->
    <template v-if="!readonly && mergeConfig.drag" slot="trigger">
      <i class="el-icon-upload"></i>
      <div class="el-upload__text">将文件拖到此处，或<em>点击上传</em></div>
    </template>
    <div v-if="mergeConfig.drag && disabled" class="disabled-mask"></div>

    <el-button
      v-if="!readonly && mergeConfig.drag && !mergeConfig.autoUpload"
      :disabled="disabled"
      class="upload-btn"
      size="small"
      type="success"
      @click="submitUpload"
    >上传到服务器</el-button>

    <!-- 不能可拖拽 是否自动上传 维度-->
    <el-button
      v-if="!readonly && !mergeConfig.drag && mergeConfig.autoUpload"
      :disabled="disabled"
      size="small"
      type="primary"
    >点击上传</el-button>

    <template v-if="!readonly && !mergeConfig.drag && !mergeConfig.autoUpload">
      <el-button
        :disabled="disabled"
        slot="trigger"
        size="small"
        type="primary"
      >选取文件</el-button>
      <el-button
        :disabled="disabled"
        class="upload-btn"
        size="small"
        type="success"
        @click="submitUpload"
      >上传到服务器</el-button>
    </template>

  </el-upload>
</template>

<style lang="sass">

  .h-layout {
    .ncform-upload {
      &.__ncform-control {
        clear: none;
        padding-top: 5px;
      }
    }
  }

  .v-layout {
    .ncform-upload {
      &.__ncform-control {
        clear: both;
      }
    }
  }

  .ncform-upload {
    position: relative;

    .disabled-mask {
      position: absolute;
      top: 0;
      left: -10px;
      width: 370px;
      height: 180px;
      cursor: not-allowed;
    }

    .upload-btn {
      margin-left: 10px;
      vertical-align: top;
    }

    &.is-read-only{
      float: left;
      .el-upload {
        display: none;
      }
    }
  }
</style>

<script>

  import ncformCommon from '@ncform/ncform-common';
  import _get from 'lodash-es/get';

  const controlMixin = ncformCommon.mixins.vue.controlMixin;

  export default {

    mixins: [controlMixin],

    props: {
      value: {
        type: Array,
        default () {
          return [];
        }
      }
    },

    data() {
      return {
        // 组件特有的配置属性
        defaultConfig: {
          uploadUrl: '', // 上传的地址
          multiple: false, // 是否支持多选
          data: {}, // 上传时附带的额外参数
          showFileList: true, // 是否显示已上传文件列表
          drag: false, // 是否启用拖拽上传
          accept: '', // 接受上传的文件类型
          listType: 'text', // 文件列表的类型。 可选值：text/picture/picture-card
          autoUpload: false, // 是否在选取文件后立即进行上传
          limit: 1, // 最大允许上传个数
          constraint: { // 约束 TODO: 未实现
            width: 0, // 图片宽度
            height: 0, // 图片高度
            sizeFixed: true, // 图片尺寸约束的大小是否按固定值，当为false时按比例
            maxSize: 0, // 最大图片大小，单位KB，0代表不限
            minSize: 0 // 最小图片大小，单位KB，0代表不限
          },
          resField: '', // 获取返回结果的字段,
          fileNameField: 'name',
          fileUrlField: 'url',
          fileField: 'file'
        },
        uploadInfo: {
          numUploaded: 0,
          total: 0,
          count: 0,
          num: {
            success: 0, // 成功个数
            error: 0  // 失败个数
          }
        }
        // mergeConfig: 请使用该值去绑定你的组件的属性，它包含了defaultConfig data和config props的值
        // modelVal：请使用该值来绑定实际的组件的model--> [{name: 'xx', url: ''}]
      }
    },

    computed: {
      // disabled / readonly / hidden / placeholder 你可以直接使用这些变量来控制组件的行为
      uploadUrl() {
        return this.mergeConfig.uploadUrl
            || this.config.uploadUrl
            || this.defaultConfig.uploadUrl;
      },
      listType() {
        const vm = this;
        const listType = vm.mergeConfig.listType;
        return listType === 'picture-card' ? '' : listType;
      },
      fileList() {
        const vm = this;
        const modelVal = vm.modelVal;
        let numUploaded = 0;
        for (let i in modelVal) {
          if (modelVal[i].status === 'success' || !(modelVal[i].status)) {
            numUploaded++;
            if (!(modelVal[i].status)) modelVal[i].status = 'success';
          }
        }
        vm.uploadInfo.numUploaded = numUploaded;
        return modelVal;
      },
      showFileList() {
        return this.readonly || this.mergeConfig.showFileList;
      }
    },

    methods: {
      submitUpload() {
        this.$refs.upload.submit();
      },
      handleUploadChange(file, fileList) {
        const vm = this;
        vm.uploadInfo.total = fileList.length - vm.uploadInfo.numUploaded;

        if (!vm.mergeConfig.autoUpload
          && !vm.mergeConfig.showFileList
          && fileList.length) {
            vm.$message({
              message: `成功选中文件！目前选中文件共 ${fileList.length} 个`,
              type: 'success'
            });
        }
      },
      handleUploadSucess(res, file, fileList) { // 每个文件回调一次
        const vm = this;
        let i;
        for (i in fileList) {
          if (fileList[i].uid === file.uid) {
            const { resField, fileUrlField, fileNameField } = vm.mergeConfig;
            const resObj = _get(res, resField || '', {});
            fileList[i].name = resObj && resObj[fileUrlField] || fileList[i].name;
            fileList[i].url = resObj && resObj[fileNameField] || fileList[i].url;
            break;
          }
        }
        if (fileList[i].url) {
          vm.uploadCallback({ state: 'success', fileList });
        } else {
          // 去掉这个fileList
          fileList.splice(i,1);
          vm.uploadCallback({ state: 'error', fileList });
        }
      },
      handleUploadError(err, file, fileList) { // 每个文件回调一次
        this.uploadCallback({ state: 'error', fileList });
      },
      uploadCallback({state, fileList}) {
        const vm = this;
        const total = vm.uploadInfo.total;
        vm.uploadInfo.count++;
        vm.uploadInfo.num[state]++;
        if ( vm.uploadInfo.count === total) { // 完成上传
          let content;
          if (total === vm.uploadInfo.num[state]) {
            vm.$message({
              message: state === 'success' ? '上传成功！' : '上传文件全部失败！',
              type: state
            });
          } else {
            vm.$message({
              message: '部分文件上传失败！',
              type: 'error'
            });
          }

          vm.modelVal = fileList;
          vm.uploadInfo.count = 0;
          vm.uploadInfo.num.success = 0;
          vm.uploadInfo.num.error = 0;
        }
      },
      handleUploadExceed(files, fileList) {
        const vm = this;
        vm.$message({
          message: `此次选择导致一次上传的文件个数超过限制数目${vm.mergeConfig.limit}个，请重新选择！`,
          type: 'warning'
        });
      },
      handleUploadRemove(file, fileList) {
        const vm = this;
        if (file.status === 'success') {
          vm.uploadInfo.numUploaded--;
        }
        vm.uploadInfo.total = fileList.length - vm.uploadInfo.numUploaded;
        vm.modelVal = fileList;
      },
    }
  }
</script>
