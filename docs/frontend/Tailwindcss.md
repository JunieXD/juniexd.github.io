---
title: Tailwindcss 常见用法
date: 2025-10-29
comments: true
author: Junie
---
## 盒模型相关

1. 宽度和高度`width` 和 `height`
	1. `w-<number>` 表示宽度， `h-<number>` 表示高度（会根据 HTML 根元素的 font-size 计算实际值）。
	2. `w-<fraction>` fraction 表示分数，比如 1 / 2 = 50% 。
	3. 也可以用`w-2xs/xs/sm/md/lg/xl/2xl` 这种表示方法。
	4. `w-auto/full/screen/min/max/fit` 这些是特殊一点的。
	5. `w-[<value>]` 可以自定义值。
2. 尺寸`size-` 是同时设置 `width` 和 `height`。
3. 最小最大宽度高度`min-width` 和 `max-width`
	`min-w-<>` 类似上面的用法，用于设定最小和最大值。
4.  背景
	`bg-<>` 背景颜色。
5. 内外边距`padding` 和 `margin` 
	1. 用 `p-<>` 和 `m-<>` 表示内外边距（上下左右都是）。
	2. 如果单独调左右用 `px-<>` ，上下用 `py-<>` 。
	3. 左 `l` ，右 `r` ，上 `t`，下 `b`，起始 `s` ，结束 `e`。
6. 边框
	1. `border-<>` 边框可以是长度或颜色。
	2. `rounded-<>` 圆角
	3. `shadow-<>` 阴影，长度和颜色。
7. 位置
	1. `absolute` 表示绝对位置。
	2. `top-<>` 距离上面，同样的`left-<>` ...

## 文字相关

1. 颜色、大小、对齐
	1. `text-<number>` 大小。
	2. `text-<color>` 颜色。
	3. `text-<center/left/right>` 对齐。
2. 行距
	`leading-<number>` 行距。
3. 粗细
	`font-<bold/light...>`
4. 字体
	`font-<sans/serif>`

## flex / grid 相关

1. `flex` 
	1. 相当于 `display: flex` 默认是横向。
	2. `flex-row` 横向，`flex-col` 纵向，`flex-row-reverse` 反向。
2. `hidden` 隐藏，相当于 `display: none`
3. `flex-<wrap/nowarp>` 溢出。
4. `basis-<number>` 默认值，配合 `grow` 进行放大（填充没用的空间）`shrink` 缩小 。
5. `grid`
	1. `grid-cols-<number>` 有多少列，`grid-rows-<number>` 多少行。
6. `justify-<>` 主轴排布，`content-<>` 纵轴排布。
7. `justify-item-<>` 、`items-<>`。
8. `gap` 调整间距。

## 封装

在 Vue 中使用示例如下。

```vue
<template>
  <h1 class="xxx">Hello world!</h1>
</template>

<style>
  @reference "../../app.css";
  .xxx {
    @apply text-2xl font-bold text-red-500;
  }
</style>
```