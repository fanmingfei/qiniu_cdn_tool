# 七牛 CDN 上传工具

> 七牛 CDN 上传工具

![](http://src.fanmingfei.com/tool/f34335570df72584da95bbce50828e73.png)

## 使用方法
- 点击右下角上传文件
- 将文件拖拽到应用内
- 截图后粘贴到应用内（目前只支持截图粘贴）
- 右下角设置图像质量

## 构建方法

### Step1
创建七牛账号和仓库，获取 accessKey/secretKey

在这里可以找得到：

![](http://src.fanmingfei.com/tool/3205b81520fa34ddd52fe0b798e3bff5.png)

### Step2
编辑 `./src/renderer/config.json`
```
{
    "accessKey": "", // 七牛的 accessKey
    "secretKey": "", // 七牛的 secretKey
    "bucket": "", // 七牛仓库名称
    "domain": "", // 该仓库绑定的域名
    "basePath": "" // 上传文件前缀
}
```

上传文件前缀：
![](http://src.fanmingfei.com/tool/3746936b0aa5f063e80bfb9acdc8353f.png)



### Step3

启动/构建

``` bash
# install dependencies
npm install

# serve with hot reload at localhost:9080
npm run dev

# build electron application for production
npm run build


```

---

This project was generated with [electron-vue](https://github.com/SimulatedGREG/electron-vue)@[8fae476](https://github.com/SimulatedGREG/electron-vue/tree/8fae4763e9d225d3691b627e83b9e09b56f6c935) using [vue-cli](https://github.com/vuejs/vue-cli). Documentation about the original structure can be found [here](https://simulatedgreg.gitbooks.io/electron-vue/content/index.html).
