<template>
  <div class="import-data" v-loading="loading">
    <div v-if="importStatus !== 'success'">
      <h3 style="margin: 10px 0 16px 0;">数据导入</h3>
      <h4>请上传外部答题数据到本问卷中</h4>
      <span class="upload-tip">
        <i class="eztest eztest-xinxi" style="color: #366fff;"></i>
        <span style="font-size: 12px;">上传的问卷内容必须与本问卷内容保持一致，否则将导致导入失败</span>
      </span>
      <div class="upload-block" v-show="!file" @dragover="fileDragOver" @dragleave="fileDragLeave" @drop="fileDrop" @click="$refs.fileUpload.click()">
        <input @change="onFileChange" accept=".xls, .xlsx" ref="fileUpload" v-show="false" type="file"/>
        <div class="btn">
          <div class="circle-btn"><i class="eztest eztest-a-shangchuan1" style="color: #313438;"></i></div>
        </div>
        <div class="desc">
          将文件拖到此处 或 <el-button type="text">选择文件</el-button>
        </div>
      </div>
      <div class="has-file" v-show="file">
        <span v-if="file">
          <svg t="1694574923904" class="icon" viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg" p-id="7663" width="18" height="22"><path d="M145.92 0C133.461333 0 120.149333 5.12 110.976 15.36 101.845333 25.6 96 38.4 96 51.2v921.6c0 12.8 4.992 26.453333 14.976 35.84 9.984 10.24 22.442667 15.36 34.944 15.36h732.16c12.501333 0 25.813333-5.12 34.944-15.36a51.2 51.2 0 0 0 14.976-35.84v-682.666667L645.12 0H145.92z" fill="#5ACC9B" p-id="7664"></path><path d="M928 290.133333h-232.96a47.36 47.36 0 0 1-34.944-15.36 49.749333 49.749333 0 0 1-14.976-35.84V0l282.88 290.133333z" fill="#BDEBD7" p-id="7665"></path><path d="M473.728 539.306667l-113.152-162.133334h78.208l74.069333 116.053334 77.354667-116.053334h75.690667l-115.626667 162.133334 121.472 172.373333h-79.018667L511.146667 588.8l-81.493334 123.733333H352.213333z" fill="#FFFFFF" p-id="7666"></path></svg>
          {{ file.name }}
        </span>
        <el-button class="delete-file" type="text" @click="resetStatus">{{ $t('删除') }}</el-button>
      </div>
      <!-- <div>
        文件上传校对中……{{  }}
      </div> -->
      <span class="upload-tip">格式要求：Excel (.xls/.xlsx) ，请参考 <el-button type="text" @click="downloadTemplate">数据格式模板</el-button></span>
      <span v-show="false">
        <h4 style="margin-top: 8px;">导入方式</h4>
        <el-radio v-model="importType" label="1" size="small">与原有数据进行叠加并去除重复项</el-radio>
        <el-radio v-model="importType" label="2" size="small">覆盖原有数据</el-radio>
      </span>
      <div class="import-data-footer">
        <el-button :disabled="!file || disAllow" @click="importFile(1)" size="small" type="primary">确定导入</el-button>
      </div>
    </div>
    <div v-else class="import-data-success">
      <img src="~assets/img/data-center/importDataSuccess.png">
      <div class="success-text">
        <h3>导入成功</h3>
        <span>{{ `已导入文档“${file.name}”，共${importResult.sample_num}条数据` }}</span>
        <el-button @click="resetStatus" plain size="small">返回继续导入</el-button>
      </div>
    </div>
    <el-dialog class="error-dialog" title="异常提示" :visible.sync="dialogVisible">
      <div class="error-info">导入失败，上传的问卷内容必须与本问卷内容保持一致。<br>{{ errorInfo }}</div>
      <div class="error-tip">注：若未按照模板填写数据，请先<el-button type="text" @click="downloadTemplate">下载模板</el-button></div>
      <span slot="footer" class="dialog-footer">
        <el-button @click="dialogVisible = false" size="small">取消</el-button>
        <el-button type="primary" @click="closeDialog" size="small">重新上传</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
  import analysisSampleApi from '@/api/analysis/sample'
  import { getTokenId } from 'Lib/utils/index'
  export default {
    data () {
      return {
        file: null,
        importType: '1',
        importStatus: 'default',
        importResult: {sample_num: 0},
        disAllow: true,
        percent: 0,
        dialogVisible: false,
        errorInfo: '',
        loading: false
      }
    },
    methods: {
      downloadTemplate () {
        let url = `//${location.host}/index.php/analysis/sample/downloadImportSampleDataTemplate?token_id=${getTokenId()}`
        window.open(url)
      },
      onFileChange () {
        if (!this.$route.query.survey_id) return this.$message.error('问卷id获取失败，请刷新或重新进入页面')
        if (!this.importType) return this.$message.error('请先选择导入方式')
        const uploadFiles = this.$refs.fileUpload.files
        if (!uploadFiles.length) return
        if (!['xls', 'xlsx'].includes(uploadFiles[0].name.slice(uploadFiles[0].name.lastIndexOf('.') + 1).toLowerCase())) {
          this.$message.warning(this.$t('只支持支持.xls，.xlsx格式'))
          return
        }
        this.importFile(0)
      },
      fileDragOver (e) {
        e.preventDefault()
        this.fileHoverClass('add')
      },
      fileDragLeave (e) {
        e.preventDefault()
        this.fileHoverClass('remove')
      },
      fileDrop (e) {
        e.preventDefault()
        this.$refs.fileUpload.files = e.dataTransfer.files
        this.fileHoverClass('remove')
        this.onFileChange()
      },
      fileHoverClass (type) {
        const uploadBox = document.querySelector('.upload-block')
        if (uploadBox) {
          uploadBox.classList[type]('file-hover')
        }
      },
      async importFile (isImport = 0) {
        try {
          this.loading = true
          const formData = new FormData()
          formData.append('survey_id', this.$route.query.survey_id)
          formData.append('import_type', this.importType)
          formData.append('is_import', isImport)
          formData.append('file', this.$refs.fileUpload.files[0])

          // 系统接口方式请求
          const res = await analysisSampleApi.importSampleData(formData)
          if (res && res.message === 'success') {
            this.file = this.$refs.fileUpload.files[0]
            this.disAllow = false
            this.$message.success(isImport ? '导入成功' : '上传成功')
            if (isImport) {
              this.importStatus = 'success'
              this.importResult = res.data
            }
          } else {
            this.disAllow = true
            this.errorInfo = res.message
            this.dialogVisible = true
            this.$refs.fileUpload.value = null
            this.file = null
            return // this.$message.error('导入失败')
          }

          // #region 当前测试表格数据需要校对较少，使用封装没问题。后续测试会加大校验数据量，可能会考虑使用异步处理。若后续没问题，该段删掉
          // xhr异步请求
          // const _this = this
          // formData.append('token_id', getTokenId())
          // const xhr = new XMLHttpRequest()
          // xhr.open('POST', location.origin + '/index.php/analysis/sample/importSampleData')
          // xhr.send(formData)
          // // xhr.upload.onprogress = function (e) {
          // //   if (e.lengthComputable) {
          // //     this.percent = Math.ceil((e.loaded / e.total) * 100)
          // //     console.log('上传进度', this.percent + '%')
          // //   }
          // // }
          // // xhr.upload.onload = function (e) {
          // //   console.log('上传完成')
          // //   this.importStatus = 'success'
          // // }
          // xhr.upload.addEventListener('progress', function (e) {
          //   if (e.lengthComputable) {
          //     this.percent = Math.ceil((e.loaded / e.total) * 100)
          //     console.log('上传进度', this.percent + '%')
          //   }
          // })
          // xhr.addEventListener('load', function (e) {
          //   console.log('上传完成')
          // })
          // xhr.onreadystatechange = function () {
          //   if (xhr.readyState === 4 && xhr.status === 200) {
          //     let xhrRes = JSON.parse(xhr.responseText)
          //     if (xhrRes.code === 0 && xhrRes.message === 'success') {
          //       console.log('上传数据', xhrRes)
          //       _this.$message.success('上传成功')
          //       if (isImport) {
          //         _this.importStatus = 'success'
          //         _this.importResult = xhrRes
          //       }
          //     } else {
          //       console.log(xhrRes)
          //       console.log('上传失败')
          //     }
          //   }
          // }
          // #endregion
        } catch (e) {
          throw e
        } finally {
          this.loading = false
          if (isImport) {
            this.$refs.fileUpload.value = null
          }
        }
      },
      resetStatus () {
        this.importStatus = 'default'
        this.$nextTick(() => {
          this.$refs.fileUpload.value = null
          this.file = null
        })
      },
      closeDialog () {
        this.dialogVisible = false
        this.errorInfo = ''
        this.$refs.fileUpload.click()
      }
    }
  }
</script>

<style lang="less" scoped>
.import-data {
  .upload-tip {
    color: #66686E;
    font-size: 14px;
    margin: 8px 0;
    display: flex;
    align-items: center;
    .el-button {
      padding: 0;
      text-decoration: underline
    }
  }
  .upload-block {
    width: 480px;
    height: 130px;
    background: #F9FAFC;
    border-radius: 8px;
    border: 1px dashed #BABCBE;
    // margin: 8px 0;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    .btn {
      i {
        font-size: 24px;
      }
    }
    .desc {
      text-align: start;
      margin-left: 10px;
      font-size: 14px;
      color: #999999;
    }
  }
  .upload-block:hover {
    background-color: rgba(54, 111, 255, .05);
    border-color: #366fff;
  }
  .file-hover {
    background-color: rgba(54, 111, 255, .1);
    border-color: #366fff;
  }
  .has-file {
    width: 480px;
    height: 56px;
    background: #F9FAFC;
    border-radius: 8px;
    border: 1px solid #E4E6E8;
    position: relative;
    cursor: unset;
    padding: 8px;
    display: flex;
    justify-content: space-between;
    align-items: center;
    i {
      margin-right: 6px;
    }
    span {
      font-size: 14px;
      color: #040508;
      text-overflow: ellipsis;
      overflow: hidden;
      word-break: break-all;
      white-space: nowrap;
      display: flex;
      align-items: center;
      svg {
        margin-right: 4px;
      }
    }
  }
  .import-data-footer {
    margin-top: 24px;
  }
  .import-data-success {
    height: 380px;
    display: flex;
    align-items: center;
    justify-content: center;
    img {
      height: 150px;
      width: 150px;
    }
    .success-text {
      display: flex;
      flex-direction: column;
      align-items: flex-start;
      span {
        color: #66686E;
        font-size: 14px;
        max-width: 320px;
        margin: 8px 0 16px 0;
      }
    }
    .el-button {
      border-color: #366fff;
      border-radius: 6px;
      color: #366fff;
    }
  }
  .error-dialog {
    /deep/.el-dialog {
      width: 480px;
      background: #FFFFFF;
      box-shadow: 0px 4px 12px 0px rgba(0,0,0,0.1), 0px 1px 4px 0px rgba(0,0,0,0.1);
      border-radius: 8px;
      border: 1px solid #E4E6E8;
      .el-dialog__header {
        .el-dialog__title {
          font-weight: bold;
        }
      }
    }
    .error-info {
      font-size: 14px;
      color: #040508;
      line-height: 22px;
    }
    .error-tip {
      font-size: 14px;
      color: #66686E;
    }
    .dialog-footer {
      .el-button {
        border-radius: 6px;
      }
    }
  }
}
</style>
