# li-clipper 图片裁剪组件

> 高性能、所见即所得的图片裁剪组件，支持 uni-app 多端，支持图片旋转、缩放、拖动、锁定宽高/比例，裁剪结果100%还原裁剪框内容，适合头像、证件照等多种场景。

---

## 说明
- uniapp插件市场上合适的裁剪组件良莠不齐，实际使用起来也是缺失好些功能，要么功能缺失，要么功能失效，要么计算有误性能达不到要求，索性自行基于原有组件进行二次改造，逐步完善，现在重新整理归档留存记录，同时也希望能够帮到后续有所需要的开发者。
- 目前仅用于H5端为主，小程序端为辅。

## 平台兼容

| H5  | 微信小程序 | 支付宝小程序 | 百度小程序 | 头条小程序 | QQ 小程序 | App |
| --- | ---------- | ------------ | ---------- | ---------- | --------- | --- |
| √   | √          | 未测         | 未测       | 未测          | 未测      | 未测   |

## 安装

放置于项目components目录下，页面import引入使用。

---

## 基本用法

```html
<template>
  <view>
    <image :src="url" v-if="url" mode="widthFix"></image>
    <li-clipper
      v-if="show"
      :image-url="imageUrl"
      :width="400"
      :height="400"
      :is-lock-ratio="true"
      :is-show-cancel-btn="true"
      :is-show-photo-btn="true"
      :is-show-rotate-btn="true"
      :is-show-confirm-btn="true"
      @success="onSuccess"
      @cancel="onCancel"
    >
      <view slot="cancel">自定义取消</view>
      <view slot="photo">自定义选择图片</view>
      <view slot="rotate">自定义旋转</view>
      <view slot="confirm">自定义确定</view>
    </li-clipper>
    <button @tap="show = true">裁剪</button>
  </view>
</template>

<script>
export default {
  data() {
    return {
      show: false,
      url: "",
      imageUrl: "",
    }
  },
  methods: {
    onSuccess(e) {
      this.url = e.url;
      this.show = false;
    },
    onCancel() {
      this.show = false;
    }
  }
}
</script>
```

---

## 主要特性

- 支持多端（H5/小程序/APP等）
- 支持图片旋转、缩放、拖动、锁定宽高/比例
- 裁剪结果100%还原裁剪框内容，所见即所得
- 支持自定义按钮/插槽
- 支持多种图片来源
- 支持裁剪框尺寸、比例、移动范围等灵活配置

---

## Props 属性

| 参数              | 说明                       | 类型     | 默认值   |
| ----------------- | -------------------------- | -------- | -------- |
| value             | 是否显示组件               | Boolean  | true     |
| image-url         | 图片路径                   | String   | -        |
| file-type         | 导出图片类型（png/jpg）    | String   | png      |
| quality           | 图片质量（0-1）            | Number   | 1        |
| width             | 裁剪框宽度                 | Number   | 400      |
| height            | 裁剪框高度                 | Number   | 400      |
| min-width         | 裁剪框最小宽度             | Number   | 200      |
| min-height        | 裁剪框最小高度             | Number   | 200      |
| max-width         | 裁剪框最大宽度             | Number   | 600      |
| max-height        | 裁剪框最大高度             | Number   | 600      |
| dest-width        | 导出图片宽度               | Number   | -        |
| dest-height       | 导出图片高度               | Number   | -        |
| is-lock-width     | 锁定裁剪框宽度             | Boolean  | false    |
| is-lock-height    | 锁定裁剪框高度             | Boolean  | false    |
| is-lock-ratio     | 锁定裁剪框比例             | Boolean  | true     |
| scale-ratio       | 导出图片清晰度比例         | Number   | 1        |
| min-ratio         | 最小缩放比例               | Number   | 0.5      |
| max-ratio         | 最大缩放比例               | Number   | 2        |
| is-disable-scale  | 禁止缩放                   | Boolean  | false    |
| is-disable-rotate | 禁止旋转                   | Boolean  | false    |
| is-limit-move     | 限制图片移动范围           | Boolean  | false    |
| is-show-photo-btn | 显示选择图片按钮           | Boolean  | true     |
| is-show-rotate-btn| 显示旋转按钮               | Boolean  | true     |
| is-show-confirm-btn| 显示确定按钮              | Boolean  | true     |
| is-show-cancel-btn| 显示取消按钮               | Boolean  | true     |
| rotate-angle      | 每次旋转角度               | Number   | 90       |
| z-index           | 组件层级                   | Number   | 99       |
| custom-style      | 自定义样式                 | String   | -        |
| source            | 图片来源配置               | Object   | {album: '从相册中选择', camera: '拍照'} |

---

## 事件 Events

| 事件名  | 说明             | 回调参数                                 |
| ------- | ---------------- | ---------------------------------------- |
| success | 裁剪成功         | {url, width, height}                     |
| fail    | 裁剪失败         | error对象                                |
| cancel  | 取消裁剪         | false                                    |
| ready   | 图片加载完成     | {width, height, path, orientation, type} |
| change  | 图片缩放/尺寸变化| {width, height}                          |
| rotate  | 图片旋转         | angle                                    |

---

## 插槽 Slots

- `cancel` 取消按钮
- `photo` 选择图片按钮
- `rotate` 旋转按钮
- `confirm` 确定按钮
- 默认插槽（可自定义工具栏等）

---

## 样式变量

- `--li-clipper-confirm-color` 控制确定按钮颜色

---

作者：angusli  
日期：2025-07-08