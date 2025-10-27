---
title: Vue 基础知识
date: 2025-10-27
comments: true
author: Junie
---
## 声明式渲染

> Vue 的核心功能是**声明式渲染**：通过扩展于标准 HTML 的模板语法，我们可以根据 JavaScript 的状态来描述 HTML 应该是什么样子的。当状态改变时，HTML 会自动更新。

### 单文件组件（Single-File Component）

Vue 的单文件组件相当于把 HTML、CSS、JavaScript 封装在**同一个 .vue 后缀的文件中**，方便进行复用。

### 响应式

> 能在改变时触发更新的状态被称作是**响应式**的。

Vue 提供了 `ref()` 和 `reactive()` 函数来**声明响应式状态**。

这两个函数会创建一个 JS Proxy 对象，比如 `const cnt = ref(0)` 得到 `cnt = {value: 0}` 这样一个 JS 对象，Vue 能够追踪 `cnt.value` 属性的变化，来更新 HTML 页面。

两者的区别，`ref()` 是追踪**值**的变化，而 `reactive()` 则是追踪**对象**的变化。

## 属性绑定

> **指令**是由 `v-` 开头的一种特殊属性。

给属性绑定一个动态的值，使用 `v-bind` 指令，可以简写为 `:` 。
例如
```vue
<div v-bind:id="dynamicId"></div>
```
 可以写成 
 ```vue
 <div :id="dynamicId"></div>
 ```

## 事件监听

Vue 使用 `v-on` 指令监听 DOM 事件，可以简写为 `@`。
例如
```vue
<button v-on:click="increment">{{ count }}</button>
```
可以写成
```vue
<button @click="increment">{{ count }}</button>
```

## 表单绑定

Vue 使用 `v-bind` 和 `v-on` 来在表单的**输入元素**上创建**双向绑定**。

```vue
<input :value="text" @input="onInput">

function onInput(e) {
  // v-on 处理函数会接收原生 DOM 事件
  // 作为其参数。
  text.value = e.target.value
}
```

Vue 使用 `v-model` 作为语法糖，可以简化为

```vue
<input v-model="text">
```

将被绑定的值与 `<input>` 的值自动同步。

## 条件渲染

`v-if` 指令可以进行条件渲染。

```vue
<h1 v-if="awesome">Vue is awesome!</h1>
```

其中 `awesome` 是一个变量，当它为真的时候这个 `h1` HTML标签才会被渲染。
当然有 `v-else-if` 和 `v-else` 也作为条件渲染指令。

```vue
<h1 v-if="awesome">Vue is awesome!</h1>
<h1 v-else>Oh no 😢</h1>
```

## 列表渲染

Vue 使用 `v-for` 指令来渲染一个基于数组的列表。

```vue
const todos = ref([
  { id: id++, text: 'Learn JavaScript' },
  { id: id++, text: 'Learn HTML' },
  { id: id++, text: 'Learn Vue' }
])

<ul>
  <li v-for="todo in todos" :key="todo.id">
    {{ todo.text }}
  </li>
</ul>
```

如果想要更新列表，可以

1. 在源数组上调用变更方法
  ```js
  todos.value.push(newTodo)
  ```
2. 使用新数组代替原数组
  ```js
  todos.value = todos.value.filter(/* ... */)
  ```

## 计算属性

`computed` 是一个自动更新的函数。输入是依赖的响应式数据，输出是一个新的响应式值。可以用来作为“派生数据”。

```js
import { ref, computed } from 'vue'

const price = ref(100)
const count = ref(2)

const total = computed(() => price.value * count.value)
```

## 生命周期和模板引用

为了手动操作 DOM，Vue 使用**模板引用**——指向模板中一个 DOM 元素的 ref。

通过同名的 ref 来访问引用，通过**生命周期钩子**在 DOM 的不同生命周期进行不同的操作。

```vue
<script setup>
import { ref, onMounted } from 'vue'

const pElementRef = ref(null)

onMounted(() => {
  pElementRef.value.textContent = 'mounted!'
})
</script>

<template>
  <p ref="pElementRef">Hello</p>
</template>
```

组件的生命周期如图所示。

![](../assets/images/生命周期图.png )

## 侦听器

Vue 用 `watch()` 来侦听数据源，比如 ref。

```js
import { ref, watch } from 'vue'

const count = ref(0)

watch(count, (newCount) => {
  console.log(`new count is: ${newCount}`)
})
```

## 组件

父组件可以在模板中渲染另一个组件作为子组件。

```js
<script setup>
import ChildComp from './ChildComp.vue'
</script>

<template>
  <ChildComp />
</template>
```

## Props

子组件可以通过 **props** 从父组件接受动态数据。

```js
// ChildComp.vue
<script setup>
const props = defineProps({
  msg: String
})
</script>
```

```vue
<ChildComp :msg="greeting" />
```

## Emits

子组件可以向父组件触发事件。

```vue
<script setup>
// 声明触发的事件
const emit = defineEmits(['response'])

// 带参数触发
emit('response', 'hello from child')
</script>
```

父组件可以使用 `v-on` 监听子组件触发的事件。

```vue
<ChildComp @response="(msg) => childMsg = msg" />
```

## 插槽

除了通过 props 传递数据外，父组件还可以通过**插槽** (slots) 将模板片段传递给子组件。

```vue
// ChildComp.vue
<template>
  <slot>Fallback content</slot>
</template>
```

```vue
<script setup>
import { ref } from 'vue'
import ChildComp from './ChildComp.vue'

const msg = ref('from parent')
</script>

<template>
  <ChildComp>Message: {{ msg }}</ChildComp>
</template>
```