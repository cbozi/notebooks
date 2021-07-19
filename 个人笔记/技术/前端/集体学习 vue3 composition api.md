集体学习 vue3 composition api

## setup

### arguments: 
- props
- context

```
export default {
  setup(props, { attrs, slots, emit }) {
    ...
  }
}
```

1、props的类型是Reactive，对props使用解构会失去相应性。
```
const { title } = props // ×
const title = props.title // ×

const { title } = toRefs(props)
cosnt title = toRef(props, 'title')
```

### setup 返回值

1、返回一个对象，则可以在模版中访问该对象的properties
注意，从 setup 返回的 refs 在模板中访问时是被自动解开的，因此不应在模板中使用 .value。
2、返回渲染函数
```
// 官方文档
// MyBook.vue

import { h, ref, reactive } from 'vue'

export default {
  setup() {
    const readersNumber = ref(0)
    const book = reactive({ title: 'Vue 3 Guide' })
    // Please note that we need to explicitly expose ref value here
    return () => h('div', [readersNumber.value, book.title])
  }
}
```

## reactivity 

### 创建响应式对象

- ref
- reactive

### computed

- 只读computed
接受一个getter函数
- 可写的computed
传入get，set
```
const count = ref(1)
const plusOne = computed({
  get: () => count.value + 1,
  set: val => {
    count.value = val - 1
  }
})
```


### watchEffect
自动收集依赖，类似没有返回值的computed

### watch
侦听数据源必需是响应式对象，或者ref
```
// 侦听一个 getter
// 如果直接去state.count因为是值类型，会失去响应性，可以传一个getter函数
const state = reactive({ count: 0 })
watch(
  () => state.count,
  (count, prevCount) => {
    /* ... */
  }
)

// 直接侦听一个 ref
const count = ref(0)
watch(count, (count, prevCount) => {
  /* ... */
})
```

## provide/inject

可以直接传递reactive已实现响应性。

注意provide/inject必须在setup方法的顶层使用。

## 使用经验

### 关于computed

tutor-web-config-generator的配置化生成表单的实现思路是，整体表单的值使用project/inject暴露给所有子组件，并且用path传入子组件对应数据所在位置，子组件直接进行修改。于是一开始写了类似这样的逻辑：

```
// template
<input v-model="value"/>
// script
const formValue = inject(GENERATION_FORM_VALUE)
const value = computed(() => {
	get(formValue, props.path)
})
```
这样的代码在运行时报错，而且输入框输入时文字不变，报错是说computed对象是不可修改的，vue3在运行时做了限制。

后来用了watchEffect
```
const value = ref()
watchEffect(() => {
	value.value = get(formValue, props.path)
})
```

用带有getter, setter的computed实现更合适
```
const value = computed({
	get: () => get(formValue, props.path),
	set: (value) => set(formValue, props.path, value) 
})
```

### 与三方库的相应性传递

代码：https://gerrit.zhenguanyu.com/c/tutor-web-config-generator/+/1334877

例如antd-vue 的[useForm api](https://2x.antdv.com/components/form-cn#components-form-demo-useForm-basic): `useForm(modelRef, rulesRef)`

使用的时候一开始传的ref，发现有问题，去看了它的代码才发现它的参数命名是modelRef，rulesRef，实际却要求接收reactive。

但是我其他地方实现都是使用ref，而且会进行整体的赋值`formValueRef.value = {...}`，就只能这样做了；

```
const antFormRef = shallowRef(useForm(formValueRef.value, rules))
watchEffect((onInvalidate) => {
        const antForm = useForm(formValueRef.value, rules)

        antFormRef.value = antForm
        formInstance?.registerAntForm(antForm)

        onInvalidate(() => {
            formInstance?.unregisterAntForm(antForm)
        })
})
```

这里还使用了一个shallowRef，由于ref是会递归地将对象转换为reactive.

### TSX

由于vue3还不支持参数的泛型，或者说它内部类型太过复杂，想要定义一类组件需要的props，最好的方法就是用functional component + TSX包一下。

```
export function wrapFieldComp(Comp: any) {
    return (props: { field: TemplateField; path: (string | number)[] }) => (
        <Comp {...props}></Comp>
    )
}
```