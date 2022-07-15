<template>
  <!-- 富文本 -->
  <div v-loading="tinymceLoading">
    <div v-show="!tinymceLoading">
      <textarea :id="tinymceId" />
    </div>
  </div>
</template>


<script>
import tinymce from "tinymce/tinymce";
import 'tinymce/icons/default'
import "tinymce/themes/silver";
import "tinymce/plugins/image";
import "tinymce/plugins/media";
import "tinymce/plugins/table";
import "tinymce/plugins/lists";
import "tinymce/plugins/wordcount";
import "tinymce/plugins/preview";
import "tinymce/plugins/code";
import "tinymce/plugins/link";
import "tinymce/plugins/advlist";
import "tinymce/plugins/codesample";
import "tinymce/plugins/fullscreen";
import "tinymce/plugins/searchreplace";
import "tinymce/plugins/autolink";
import "tinymce/plugins/directionality";
import "tinymce/plugins/visualblocks";
import "tinymce/plugins/visualchars";
import "tinymce/plugins/template";
import "tinymce/plugins/charmap";
import "tinymce/plugins/nonbreaking";
import "tinymce/plugins/insertdatetime";
import "tinymce/plugins/autosave";
import "tinymce/plugins/autoresize";
import "tinymce/models/dom";
import "./plugins/importcss/plugin.min"
import "./plugins/pagebreak/plugin.min"
import "./plugins/anchor/plugin.min"
import "./plugins/emoticons/plugin.min"
import "./plugins/emoticons/js/emojiimages.min"
import "./plugins/emoticons/js/emojis.min"
import zh from './langs/zh-Hans'
import plugins from './plugins'
import toolbar from './toolbar'
import OSS from "ali-oss";

export default {
  components: {},
  props: {
    id: {
      type: String,
      default: function () {
        return 'vue-tinymce-' + +new Date() + ((Math.random() * 1000).toFixed(0) + '')
      }
    },
    value: {
      type: String,
      default: ""
    },
    disabled: {
      type: Boolean,
      default: false
    },
    // 菜单
    menubar: {
      type: String,
      default: 'file edit view insert format tools table'
    },
    height: {
      type: [Number, String],
      required: false,
      default: 360
    },
    width: {
      type: [Number, String],
      required: false,
      default: 'auto'
    },
    plugins: {
      type: [String, Array],
      default: ''
    },
    toolbar: {
      type: [String, Array],
      default: ''
    }
  },
  data() {
    return {
      tinymceLoading: true,
      hasChange: false,
      hasInit: false,
      tinymceId: this.id,
      fullscreen: false,
      // oss存储参数
      authCode: {
        region: '',
        securityToken: '',
        accessKeyId: '',
        accessKeySecret: '',
        bucket: '',
        pathPrefix: ''
      },
    };
  },
  mounted() {
    this.initTinymce()
    this.getCode()
  },
  activated() {
    if (tinymce) {
      this.initTinymce()
    }
  },
  beforeDestroy() {
    this.destroyTinymce()
  },
  methods: {
    initTinymce() {
      const _this = this
      //初始化配置
      tinymce.init({
        selector: `#${this.tinymceId}`,
        plugins: plugins,
        menubar: menubar,// 菜单
        toolbar: toolbar,
        height: this.height,
        language_url: zh,// 中文
        language: 'zh-Hans',
        // 初始化数据
        init_instance_callback: editor => {
          if (_this.value) {
            editor.setContent(_this.value)
          }
          _this.hasInit = true
          editor.on('NodeChange Change KeyUp SetContent', () => {
            this.hasChange = true
            this.$emit('input', editor.getContent())
          })
        },
        // 图片使用oss
        images_upload_handler: (blobInfo, progress) => new Promise((resolve, reject) => {
          this.getBlob(blobInfo.blob()).then(res => {
            resolve(res.url);
          }).catch(err => {
            reject(err)
          })
        }),
      });
      setTimeout(() => {
        this.tinymceLoading = false
      }, 1000)
    },
    destroyTinymce() {
      // 关闭更多弹窗工具栏
      let tox = document.getElementsByClassName('tox-tinymce-aux')
      while (tox.length > 0) {
        tox[0].remove()
      }
      // if (tinymce) {
      //   const tinymce = tinymce.get(this.tinymceId)
      //   if (this.fullscreen) {
      //     tinymce.execCommand('mceFullScreen')
      //   }
      //   if (tinymce) {
      //     tinymce.destroy()
      //   }
      // }
    },
    setContent(value) {
      tinymce.get(this.tinymceId).setContent(value)
    },
    getContent() {
      tinymce.get(this.tinymceId).getContent()
    },
    // oss 参数
    getCode() {
      this.$http.post('xxx', {
        modCode: "project",
      }, {
        emulateJSON: true
      }).then(res => {
        if (res.data.code === 0) {
          this.authCode = res.data.data;
        }
      });
    },
    // 获取处理为blob格式
    async getBlob(file) {
      let base64
      await new Promise((resolve) => {
        this.getPictureBase64(file).then(res => {
          base64 = res
          resolve(res)
        })
      })
      let arr = base64.split(',');
      //注意base64的最后面中括号和引号是不转译的
      let _arr = arr[1].substring(0, arr[1].length - 2)
      let mime = arr[0].match(/:(.*?);/)[1]
      let bstr = atob(_arr)
      let n = bstr.length
      let u8arr = new Uint8Array(n)
      while (n--) {
        u8arr[n] = bstr.charCodeAt(n);
      }
      let blob = new Blob([u8arr], {type: mime})
      return this.updateFile(blob, file)
    },
    // 图片文件转为 base64
    getPictureBase64(file) {
      return new Promise((resolve, reject) => {
        const reader = new FileReader()
        reader.readAsDataURL(file)
        reader.onload = () => resolve(reader.result)
        reader.onerror = (error) => reject(error)
      });
    },
    // oss上传
    updateFile(blob, file) {
      let client = new OSS({
        region: this.authCode.region,
        // 重要配置参数
        secure: true,
        stsToken: this.authCode.securityToken,
        accessKeyId: this.authCode.accessKeyId,
        accessKeySecret: this.authCode.accessKeySecret,
        bucket: this.authCode.bucket,
      });
      const storeAs = this.authCode.pathPrefix + "/" + file.name
      return client.put(storeAs, blob, {type: blob.type})
    },
  },
  watch: {
    value(val) {
      if (tinymce) {
        if (!this.hasChange && this.hasInit) {
          this.$nextTick(() =>
              tinymce.get(this.tinymceId).setContent(val || '')
          )
        }
      }
    },
  }
}
;
</script>
<style scoped>
@import 'skins/ui/oxide/content.inline.min.css';
@import 'skins/ui/oxide/content.min.css';
@import 'skins/ui/oxide/skin.min.css';
@import 'skins/ui/oxide/skin.shadowdom.min.css';
</style>