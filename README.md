# 富文本

## 实例

```vue

<template>
  <div>
    <el-form-item label="富文本">
      <tinymce v-model="form.content"/>
    </el-form-item>
  </div>
</template>
<script setup>
import {reactive} from 'vue'
import Tinymce from '@/xxx/tinymce/index'

const form = reactive({
  content: ''
})
</script>
```