vue3 注解与调研

vue3语法向下兼容 --》 只需要简单的调整，可以随时可以切换3.0
vue3逻辑上是将原来的内部函数进行了暴露 --》 放开约束，更灵活

## 1. 初始化  -- 函数式编程
Vue2 : 面向对象
Vue3 : 函数式编程

故，在初始化时，vue的逻辑将使用函数的方式进行初始化
```javascript
// 1. 菊花链
Vue.createApp({
    data() {
        return {
            input: '# hello'
        }
    }
}).mount('#app')

// 2. 缓存
const app = Vue.createApp({
    data() {
        return {
            input: '# hello'
        }
    }
})
app.mount('#app')
```

## 2. 如何拆分.vue
官方文档如下，https://v3.vuejs.org/guide/composition-api-introduction.html#why-composition-api 此处仅为个人见解


vue2 基于面向对象的形式，将依赖属性和生命周期放在view中，也就是`.vue文件`中

在比较复杂的业务中，一个常见的问题是，如何优雅的拆分/组合.vue 【一个.vue 上千行，很常见，mixins可以解决，但很蛋疼】

注：此处并非说如何使用组件，而是说，.vue作为一个业务组合的地方，如何组合所依赖的参数，尤其是互联网以周为单位进行迭代，在组合的位置进行大规模调整，将会产生很大的风险隐患

就问题而言，分两类

- methods/data 混合

最常见的组合问题，如简单的toDoList 【https://codesandbox.io/s/github/vuejs/vuejs.org/tree/master/src/v2/examples/vue-20-todomvc?from-embed】中

如果将toDoList理解为多期迭代，第一期可显示，第二期可增加，第三期可删除。。。

第一期 - 查询：修改view，添加data【todos】，添加computed 【filteredTodos】
第二期 - 添加：修改view，添加data【newTodo】，添加methods　【addTodo】
。。。


因为位置分散，要不了多久，将无法分清需求版本，最终在删除需求时，可能也只是视图中删除

vue2：mixins，但mixins类似于c的include 【amd，cmd同期的labelModule，不过git已经删了】，属于全局的混合，类似于无节操的继承【多重】，需要通过命名的方式解决冲突或明确引入来源

vue3：api组合

- 生命周期依赖

比如最常见的，前端同步服务器时间问题

对于该时间来说，他可以是一个属性/方法 【逻辑上可以在任何地方使用，比如vuex中】，通过 `impoert` 获取即可，不需要关心更多的操作，但问题就在于`setTimeout/setInterval`在非激活状态下，执行间隔超过5s【浏览器】

一种简单的处理方式是在页面激活时，重新同步时间，但问题在于，在vuex中使用的属性，却需要在`.vue文件`中实现更新【依然可以通过mixin进行简化】

注：假设通过`import`返回的是服务器实时时间【vue.observable】，目的是为了在业务中可直接使用，不需要做过多的设置，但为了应对各种问题，依然需要在.vue中还需要如下处理

1. 在激活状态下，重新请求【fetch】服务器时间，而后在通过setInterval进行倒计时
2. 在组件摧毁/不再使用时间，的情况下，clearInterval

vue2：
1. 一种方式为：提供两类返回值，一种给需要的业务组件使用 `mixins:[serverTime]`,用以刷新，取消，一种直接使用
2. 另一种方式为：创建全局【EventBus】的观察者进行解决 【所有组件 extends 基础组件，或直接全局注入】,将vue组件的生命周期中，发射触发，在其他的地方进行监听【如vuex中】

vue3：组合生命周期 【应该需要额外操作】

显然组合api提供了一种可能

## 3. setup
组合data/methods 的入口函数，其执行时机【生命周期】在beforeCreate之前 

相关生命周期内容：初始化 --》beforeCreate --》 reactivity【响应式】 --》create --》 mount【挂载】

显然在追加data/methods时机，必须在`reactivity`之前，可能在初始化之前

## 4. reactivity --> reactive/readonly/shallowReactive/shallowReadonly
对属性一一设置级别，这让我想起了ko
```javascript
var SimpleListModel = function(items) {
    this.items = ko.observableArray(items);
    this.itemToAdd = ko.observable("");
    this.addItem = function() {
        if (this.itemToAdd() != "") {
            this.items.push(this.itemToAdd()); // Adds the item. Writing to the "items" observableArray causes any associated UI to update.
            this.itemToAdd(""); // Clears the text box, because it's bound to the "itemToAdd" observable
        }
    }.bind(this);  // Ensure that "this" is always this view model
};
 
ko.applyBindings(new SimpleListModel(["Alpha", "Beta", "Gamma"]));
```

总之`Basic Reactivity`系列，就是这么一个产物

- reactive

按照`vue.observable`传统，先给安排两类，一类数值 【不处理】，一类对象 【转换为proxy】

注：对于对象嵌套问题，reactive依然是嵌套proxy

- readonly

多人维护时，对属性的可写要求十分高，语法上为`const`,不过无法进行引用 【无法跨越】

```javascript
const aData = {a:1}
const data = {aData}
data.aData.a = 3 // 可修改，等同于aData.a = 3
data.aData = {b:1} // 可修改，aData指向修改，不涉及内容

data = {c:1} // 唯一不可修改的地方，但data的生命周期-创建，不一定在本函数中，一旦为引用，语义限制消失
```

另一个问题是产生Error，会中断程序【警告就够了】
当然，放到vue3中的一个好处则是不需要监听

- shallowReactive/shallowReadonly

shallow --》 浅  ，类似于浅copy，指定义第一层
理论应该是来自json并非“标准的结构”，使得语义和Proxy实现有一定差异

注：
```javascript
const data  = {a:{b:1}}
const proxy = new Proxy(data,{
    get(target, key){
        console.log(target, key)
        return {}
    }
})

proxy['a.b']  // key === 'a.b'
proxy.a.b   //  key === 'a' ，而后根据返回值，在请求.b【自定义嵌套】
```

## 5. reactive/shallowReactive/readonly 对性能的影响
简单的做一个循环增加修改，查看三者对内存和点击响应及执行深度的情况

- 内存

很好理解，毕竟readonly根本不需要复制任何属性

reactive：97.5m
shallowReactive：70.4m
readonly  ： 62.3m

- 响应时间  -- click

reactive： 560ms
shallowReactive：253ms
readonly  ： 193ms

- 深度 -- 图
![reactive](/static/1.png)

reactive：很深，需要操作视图，包含vnode等计算

![shallowReactive](/static/2.png)


shallowReactive：数组，修改第一个元素，整体执行深度与readonly相似

这里使用的是数组的例子，作为浅层响应式，数组内容不响应

![shallowReactive](/static/4.png)

shallowReactive：很深，修改key==0的元素，整体执行深度与readonly相似

![readonly](/static/3.png)

shallowReactive：很浅，完全不修改


## 6. ref/shallowRef
在模板中有一个问题，无法获取数据的父类【指data的指向】 ，使的无法更灵活的遍历

比如后台模板中，需要将userInfo信息，注入到js/html中，假设数据结构如下
```javascript
{
    data(){
        return {
            name:'',
            age:'',
            address:''
        }
    }
}
```
因为数据本身不依赖顺序，一个for循环的事，但因为无法获取父节点 - - 

另一个则是proxy不支持原始属性的问题

ref简单的说就是`reactive({value})`

对于value === 值时，RefImpl 代替proxy的作用
对于value === 对象时，value中为proxy对象，进行嵌套