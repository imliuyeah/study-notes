# amisRequire 导入对应模块
通过 `amisRequire` 能够引入的模块：
* amis
* amis/embed
* amis@2.8.0
* amis@2.8.0/embed
* axios
* classnames
* echarts
* exceljs
* history
* immutability-helper
* mobx
* mobx-state-tree
* moment
* papaparse
* path-to-regexp
* prop-types
* qs
* react
* react-cropper
* react-dom
* react-dropzone
* react-transition-group
* sortablejs
* zrender

使用方法如下：
```javascript
const amis = window.amisRequire('amis')
const amisEmbed = window.amisRequire('amis/embed')
```

# registerAction 注册自定义动作
```javascript
const amis = window.amisRequire('amis')
const { registerAction } = amis
const action = {
  run (action, renderer, event) {
    // 你的动作逻辑
    // ...
  }
}
registerAction('my-action', action)
```