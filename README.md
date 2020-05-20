# 4.22的vue3-beta直播演讲笔记记录

### vue3.0 diff算法改进

```html
<div>
  <span>status</span>
  <span>status</span>
  <span>status</span>
  <span>status</span>
  <span>status</span>
  <div>
    <span>123</span>
    <span :id="id" class="class1">{{ msg }}</span>
  </div>
  <span>status</span>
  <span>status</span>
  <span>status</span>
  <span>status</span>
  <span>status</span>
  <span>status</span>
  </div>
```
```typescript
import { createVNode as _createVNode, toDisplayString as _toDisplayString, openBlock as _openBlock, createBlock as _createBlock } from "vue"

export function render(_ctx, _cache) {
  return (_openBlock(), _createBlock("div", null, [
    _createVNode("span", null, "status"),
    _createVNode("span", null, "status"),
    _createVNode("span", null, "status"),
    _createVNode("span", null, "status"),
    _createVNode("span", null, "status"),
    _createVNode("div", null, [
      _createVNode("span", null, "123"),
      _createVNode("span", {
        id: _ctx.id,
        class: "class1"
      }, _toDisplayString(_ctx.msg), 9 /* TEXT, PROPS */, ["id"])
    ]),
    _createVNode("span", null, "status"),
    _createVNode("span", null, "status"),
    _createVNode("span", null, "status"),
    _createVNode("span", null, "status"),
    _createVNode("span", null, "status"),
    _createVNode("span", null, "status")
  ]))
}

```

当需要更新的时候，其他的span都不管，直接跳到带标记的span那里，对比TEXT里面文字内容的变动。
如果在默认的diff算法中，会把每个span对比一遍而且要看旧的和新的dom有没有变化，虽然js速度很快，
但是随着应用的增大，将会不可避免地占用越来越多的时间。

如果在当前的算法上，直接只看里面有没有带动态的东西，然后直接把带动态的东西过一遍行了，这样的话，
不管block嵌套得再深，仅需要通过最外层block来寻找到span，不会通过其他根本不会变的东西再遍历一遍。

而且如果有静态的id绑定，对于runtime来说，这个id在与不在都是没有区别的。
如果做成动态的id绑定，这个节点不光有Props的变化，而且还有文字的变化，["id"]，告诉模板引擎props，仅有id会变化，diff只管props中的id有没有变化，不管其他属性。

总的来说，动态更新只会关注那些真正变化的东西，跳出visual dom的更新瓶颈，又保持了可以手写render function的灵活性。

### hoistStatic
```html
<div>
  <span>status</span>
  <span>status</span>
  <span>status</span>
  <span>status</span>
  <span>status</span>
  <div>
    <span>123</span>
    <span :id="id" class="class1">{{ msg }}</span>
  </div>
  <span>status</span>
  <span>status</span>
  <span>status</span>
  <span>status</span>
  <span>status</span>
  <span>status</span>
  </div>
```

```TypeScript
import { createVNode as _createVNode, toDisplayString as _toDisplayString, openBlock as _openBlock, createBlock as _createBlock } from "vue"

const _hoisted_1 = /*#__PURE__*/_createVNode("span", null, "status", -1 /* HOISTED */)
const _hoisted_2 = /*#__PURE__*/_createVNode("span", null, "status", -1 /* HOISTED */)
const _hoisted_3 = /*#__PURE__*/_createVNode("span", null, "status", -1 /* HOISTED */)
const _hoisted_4 = /*#__PURE__*/_createVNode("span", null, "status", -1 /* HOISTED */)
const _hoisted_5 = /*#__PURE__*/_createVNode("span", null, "status", -1 /* HOISTED */)
const _hoisted_6 = /*#__PURE__*/_createVNode("span", null, "123", -1 /* HOISTED */)
const _hoisted_7 = /*#__PURE__*/_createVNode("span", null, "status", -1 /* HOISTED */)
const _hoisted_8 = /*#__PURE__*/_createVNode("span", null, "status", -1 /* HOISTED */)
const _hoisted_9 = /*#__PURE__*/_createVNode("span", null, "status", -1 /* HOISTED */)
const _hoisted_10 = /*#__PURE__*/_createVNode("span", null, "status", -1 /* HOISTED */)
const _hoisted_11 = /*#__PURE__*/_createVNode("span", null, "status", -1 /* HOISTED */)
const _hoisted_12 = /*#__PURE__*/_createVNode("span", null, "status", -1 /* HOISTED */)

export function render(_ctx, _cache) {
  return (_openBlock(), _createBlock("div", null, [
    _hoisted_1,
    _hoisted_2,
    _hoisted_3,
    _hoisted_4,
    _hoisted_5,
    _createVNode("div", null, [
      _hoisted_6,
      _createVNode("span", {
        id: _ctx.id,
        class: "class1"
      }, _toDisplayString(_ctx.msg), 9 /* TEXT, PROPS */, ["id"])
    ]),
    _hoisted_7,
    _hoisted_8,
    _hoisted_9,
    _hoisted_10,
    _hoisted_11,
    _hoisted_12
  ]))
}
```

把静态的不会动的节点，提升出去，在应用启动的时候会创建一次，然后虚拟节点，在每次渲染的时候会被不停地复用。
在大应用中，对于内存有一个很大的改进，因为不需要在每次更新创建新的visual，把旧的给销毁掉。

### 事件侦听器缓存
```html
<div>
  <span @click="onClick">static</span>
  </div>
```
```TypeScript
import { createVNode as _createVNode, openBlock as _openBlock, createBlock as _createBlock } from "vue"

export function render(_ctx, _cache) {
  return (_openBlock(), _createBlock("div", null, [
    _createVNode("span", { onClick: _ctx.onClick }, "static", 8 /* PROPS */, ["onClick"])
  ]))
}
```

在侦听v-on事件的时候，是不知道click函数是否会变化的，需要看成是一个动态的绑定，比如click函数是在data里面返回的，后面要把这个onclick给替换掉，
实际上是需要一次更新的。
```html
<div>
  <span @click="onClick">static</span>
<span @click="() => foo()">static2</span>
  </div>
```
```TypeScript
import { createVNode as _createVNode, openBlock as _openBlock, createBlock as _createBlock } from "vue"

export function render(_ctx, _cache) {
  return (_openBlock(), _createBlock("div", null, [
    _createVNode("span", {
      onClick: _cache[1] || (_cache[1] = $event => (_ctx.onClock($event)))
    }, "static"),
    _createVNode("span", {
      onClick: _cache[2] || (_cache[2] = () => _ctx.foo())
    }, "static")
  ]))
}

// Check the console for the AST
```
但是如果使用cacheHandless，会变成内联函数，第一次渲染的时候会缓存起来，后续的更新就会调用这个内联函数，因为是同一个函数，所以就没有更新的必要，所以现在就可以被看作是静态的。
在JSX中，如果OnClick写上内联函数，则每次渲染都会被更新，使用了cacheHandless的话，就相当于静态，不会被更新。
如果span是一个组件Foo，如果onclick是内联函数，父组件更新，子组件一定会更新。

```TypeScript
// 没有开启cacheHandless的情况下
import { createVNode as _createVNode, openBlock as _openBlock, createBlock as _createBlock } from "vue"

export function render(_ctx, _cache) {
  return (_openBlock(), _createBlock("div", null, [
    _createVNode("span", { onClick: _ctx.onClock }, "static", 8 /* PROPS */, ["onClick"]),
    _createVNode("span", {
      onClick: () => _ctx.foo()
    }, "static", 8 /* PROPS */, ["onClick"])
  ]))
}
```

### SSR
```html
<div>
  <span>hello</span>
  <span>hello</span>
  <span>hello</span>
  <span>hello</span>
  <span>hello</span>
  <div :id="mid">static</div>
  </div>
```
```TypeScript
import { ssrRenderAttr as _ssrRenderAttr } from "@vue/server-renderer"

export function ssrRender(_ctx, _push, _parent) {
  _push(`<div><span>hello</span><span>hello</span><span>hello</span><span>hello</span><span>hello</span><div${_ssrRenderAttr("id", _ctx.mid)}>static</div></div>`)
}
```
如果有一堆静态dom，在服务器中尽可能是字符串，极大提高服务器渲染。

### 静态节点嵌入过深的优化（hoistStatic优化）
```html
<div>
  <div>
    <span>hello</span>
    <div>
      <span>hello</span>
      <div>
        <span>hello</span>
        <div>
          <span>hello</span>
          <div>
            <span>hello</span>
            <div>
              <span>hello</span>
              <div>
                <span>hello</span>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
  <div :id="mid">static</div>
  </div>
```
```TypeScript
import { createVNode as _createVNode, createStaticVNode as _createStaticVNode, openBlock as _openBlock, createBlock as _createBlock } from "vue"

const _hoisted_1 = /*#__PURE__*/_createStaticVNode("<div><span>hello</span><div><span>hello</span><div><span>hello</span><div><span>hello</span><div><span>hello</span><div><span>hello</span><div><span>hello</span></div></div></div></div></div></div></div>", 1)

export function render(_ctx, _cache) {
  return (_openBlock(), _createBlock("div", null, [
    _hoisted_1,
    _createVNode("div", { id: _ctx.mid }, "static", 8 /* PROPS */, ["id"])
  ]))
}
```

这种情况下直接创建innerHtml，而不需要创建一堆对象把它一个一个渲染出来。

### Tree-shaking
```html
```
当前html为空

```TypeScript
export function render(_ctx, _cache) {
  return null
}

// Check the console for the AST
```
可以看到没有引入任何关于vue的东西。当你需要用到的时候，才会打包进去（webpack tree-shaking）。但是最基本的东西会被引入进去（最低限度），比如visual dom的更新算法、响应式更新，这两个是无论如何都会被
包含进去的。被去掉的可以有v-model功能。

```html
<input v-model="foo"/>
<input v-model="foo" type="checkbox"/>
<transition></transition>
```

```typescript
import { vModelText as _vModelText, createVNode as _createVNode, withDirectives as _withDirectives, vModelCheckbox as _vModelCheckbox, Transition as _Transition, Fragment as _Fragment, openBlock as _openBlock, createBlock as _createBlock } from "vue"

export function render(_ctx, _cache) {
  return (_openBlock(), _createBlock(_Fragment, null, [
    _withDirectives(_createVNode("input", {
      "onUpdate:modelValue": $event => (_ctx.foo = $event)
    }, null, 8 /* PROPS */, ["onUpdate:modelValue"]), [
      [_vModelText, _ctx.foo]
    ]),
    _withDirectives(_createVNode("input", {
      "onUpdate:modelValue": $event => (_ctx.foo = $event),
      type: "checkbox"
    }, null, 8 /* PROPS */, ["onUpdate:modelValue"]), [
      [_vModelCheckbox, _ctx.foo]
    ]),
    _createVNode(_Transition)
  ], 64 /* STABLE_FRAGMENT */))
}

// Check the console for the AST
```
可以看到当html使用v-model的时候，这里会多引入了一个_vModelText，如果改成checkBox，将会引入vModelCheckBox，使用transition的时候,将会引入_Transition，很简单的道理，没有被引用的东西，就会被tree-shaking掉。
如果只写一个hello worle，那最终打包出来的大小是13.5kb。还有一个选择性的开关，可以选择对vue2.0 api的支持，但是这个默认是不会被启用的。因为无法分析在运行时你运用到了哪些东西（比如使用options中的api），
这样的话就意味着，有一部分option API的代码是无论如何都无法tree-shaking的，如果选择tree-shaking掉，那么最小体积可以达到11.75kb。如果所有可选的运行时的东西全部加进来是22.5kb的大小。

### Composition API
composition-api.vuejs.org 详细介绍了composition-api有什么用。这是一个新的api,不会影响原来的api使用，甚至可以和原来的api一起使用。第三方库，特别是逻辑库，提供可复用逻辑的时候，尽量使用composition API去提供，
这个灵活度都会比选项的灵活度要高很多。

### Fragments
vue的模板以前是需要一个dom来包括起来的，比如div,但是现在不用这样做了，在vue3里面，可以是文字可以是多个节点，都是可以的。
```html
a
<span>a</span>
```
```TypeScript
import { createVNode as _createVNode, createTextVNode as _createTextVNode, Fragment as _Fragment, openBlock as _openBlock, createBlock as _createBlock } from "vue"

const _hoisted_1 = /*#__PURE__*/_createTextVNode("a ")
const _hoisted_2 = /*#__PURE__*/_createVNode("span", null, "a", -1 /* HOISTED */)

export function render(_ctx, _cache) {
  return (_openBlock(), _createBlock(_Fragment, null, [
    _hoisted_1,
    _hoisted_2
  ], 64 /* STABLE_FRAGMENT */))
}
```
如果是直接使用render函数的，可以直接使用一个数组，会自动变成一个碎片（_Fragment）

### 组件Teleport
可以接受一个disable的参数，比如屏幕宽度，当屏幕宽的时候，该组件可以在外面显示，当窄的时候可以又回到这层树里面。可以多个Teleprot添加内容到container里面。

### 组件Suspense
（异步加载的组件，在打包的时候，会打包成单独的js文件存储在static/js文件夹里面，在调用时使用ajax请求回来插入到html中。）
Suspense可以把一个嵌套的渲染树渲染到屏幕之前，在内存里面先渲染，在渲染的过程中会记录所有存在异步依赖的组件，当所有异步组件都被resolve的之后，才会渲染整棵树到dom去。
异步依赖可以结合Composition API可以利用一个叫async setup()的选项，如果组件中有一个async setup的函数，那么该组件会被看作是一个异步组件，相当于实现了一红嵌套的异步调动，这是在之前是完全没有办法做的。


### TypeScript
vue3是用ts重写的，不管用ts还是js都是有好处（vscode自动补全，类型声明，参数提示）。支持TSX，可以获得props自动保存类型检查等等。
如果喜欢class component的体验（vue-property-decorator）,还是会支持的，但是不推荐。

### 实验性功能
比如ts的插件，可以在模板里面获得类型检查，自动补全（@znck vscode插件），原理都是把模板转换成ts，再把ts转换成模板里去。

自定义渲染器API，全局可以使用Vue.render。（Vugel webgl渲染引擎，可以在普通vue应用里面嵌入一个单文件组件，里面用Vue的语法去表达webgl的渲染逻辑，渲染出来的东西是可以正常嵌入在vue应用里面）


