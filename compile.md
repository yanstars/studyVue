## 版本

### Runtime + Compiler

> 包含编译代码的，可以把编译过程放在运行时做，

### Runtime only

> 不包含编译代码的，需要借助 webpack 的 vue-loader 事先把模板编译成 render 函数。
> 因为少了 Compiler 部分，所以 vue 体积更小

## Runtime + Compiler 编译入口

### render

> src/platforms/web/entry-runtime-with-compiler.js

```js
const idToTemplate = cached(id => {
  const el = query(id)
  return el && el.innerHTML
})

const { render, staticRenderFns } = compileToFunctions(
  template,
  {
    outputSourceRange: process.env.NODE_ENV !== 'production',
    shouldDecodeNewlines,
    shouldDecodeNewlinesForHref,
    delimiters: options.delimiters,
    comments: options.comments,
  },
  this
)

options.render = render
options.staticRenderFns = staticRenderFns

Vue.compile = compileToFunctions
```

### compileToFunctions

> src/platforms/web/compile/index.js

```js
import { baseOptions } from './options'
import { createCompiler } from 'compiler/index'
const { compile, compileToFunctions } = createCompiler(baseOptions)
export { compile, compileToFunctions }
```

### createCompiler

> src/compile/index.js

```js
import { parse } from './parser/index'
import { optimize } from './optimizer'
import { generate } from './codegen/index'
import { createCompilerCreator } from './create-compiler'

function baseCompile(template: string, options: CompilerOptions): CompiledResult {
  const ast = parse(template.trim(), options)
  if (options.optimize !== false) {
    optimize(ast, options)
  }
  const code = generate(ast, options)
  return {
    ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns,
  }
}

export const createCompiler = createCompilerCreator(baseCompile)
```

### createCompilerCreator

> src/compile/create-compiler.js

```js
export function createCompilerCreator(baseCompile: Function): Function {
  return function createCompiler(baseOptions: CompilerOptions) {
    function compile(template: string, options?: CompilerOptions): CompiledResult {
      const finalOptions = Object.create(baseOptions)
      finalOptions = extend(baseOptions)
      const compiled = baseCompile(template.trim(), finalOptions)
      return compiled
    }

    return {
      compile,
      compileToFunctions: createCompileToFunctionFn(compile),
    }
  }
}
```

### createCompileToFunctionFn

> src/compile/to-function.js

```js
export function createCompileToFunctionFn(compile: Function): Function {
  const cache = Object.create(null)

  return function compileToFunctions(template: string, options?: CompilerOptions, vm?: Component): CompiledFunctionResult {
    options = extend({}, options)
    const warn = options.warn || baseWarn
    delete options.warn

    // check cache
    const key = options.delimiters ? String(options.delimiters) + template : template
    if (cache[key]) {
      return cache[key]
    }

    // compile
    const compiled = compile(template, options)

    // turn code into functions
    const res = {}
    const fnGenErrors = []
    res.render = createFunction(compiled.render, fnGenErrors)
    res.staticRenderFns = compiled.staticRenderFns.map(code => {
      return createFunction(code, fnGenErrors)
    })

    return (cache[key] = res)
  }
}
```

## 解析 AST

> const ast = parse(template.trim(), options)

```js
import { parseHTML } from './html-parser'
export function parse(template: string, options: CompilerOptions): ASTElement | void {
  getFnsAndConfigFromOptions(options) // 从 options 中获取方法和配置

  parseHTML(template, {
    // 解析 HTML 模板
    // options ...
    start(tag, attrs, unary) {
      let element = createASTElement(tag, attrs) // ，创建 AST 元素
      processElement(element) // 处理 AST 元素 指令  attrs
      treeManagement() // AST 树管理 父子关系
    },

    end() {
      treeManagement()
      closeElement()
    },

    chars(text: string) {
      handleText()
      createChildrenASTOfText()
    },
    comment(text: string) {
      createChildrenASTOfComment()
    },
  })
  return astRootElement
}
```

## 优化 AST

> if (options.optimize !== false) && optimize(ast, options)

```js
export function optimize(root, options) {
  if (!root) return
  // 标记静态节点
  markStatic(root)
}

function isStatic(node) {
  // type 为 1 表示是普通元素，为 2 表示是表达式，为 3 表示是纯文本。
  if (node.type === 2) {
    // 表达式，返回 false
    return false
  }
  if (node.type === 3) {
    // 静态文本，返回 true
    return true
  }
  // 此处省略了部分条件
  return !!(
    (
      !node.hasBindings && // 没有动态绑定
      !node.if &&
      !node.for && // 没有 v-if/v-for
      !isBuiltInTag(node.tag) && // 不是内置组件 slot/component
      !isDirectChildOfTemplateFor(node) && // 不在 template for 循环内
      Object.keys(node).every(isStaticKey)
    ) // 非静态节点
  )
}

function markStatic(node) {
  node.static = isStatic(node)
  if (node.type === 1) {
    // 如果是元素节点，需要遍历所有子节点
    for (let i = 0, l = node.children.length; i < l; i++) {
      const child = node.children[i]
      markStatic(child)
      if (!child.static) {
        // 如果有一个子节点不是静态节点，则该节点也必须是动态的
        node.static = false
      }
    }
  }
}
```

> 标记静态节点有两个好处：
> 一、每次重新渲染的时候不需要为静态节点创建新节点，也就是静态节点的解析器不需要重新创建
> 二、在 Virtual DOM 中 patching 的过程可以被跳过
> 优化器的实现原理主要分两步：
> 一、用递归的方式将静态节点添加 static 属性，用来标识是不是静态节点
> 二、标记所有静态根节点(子节点全是静态节点就是静态根节点)

## 生成 Code

> const code = generate(ast, options)

```js
export function generate(ast, options) {
  const state = new CodegenState(options)
  const code = ast ? genElement(ast, state) : '_c("div")'
  return {
    render: `with(this){return ${code}}`,
    staticRenderFns: state.staticRenderFns,
  }
}

export function genElement(el, state) {
  let code
  const data = genData(el, state)
  const children = genChildren(el, state, true)
  code = `_c('${el.tag}'${
    data ? `,${data}` : '' // data
  }${
    children ? `,${children}` : '' // children
  })`
  return code
}
```

### mount

> $mount 定义在 platforms/runtime/index.js

```js
import { mountComponent } from 'core/instance/lifecycle'

Vue.prototype.$mount = function (el?: string | Element, hydrating?: boolean): Component {
  el = el && inBrowser ? query(el) : undefined
  return mountComponent(this, el, hydrating)
}

export function mountComponent(vm: Component, el: ?Element, hydrating?: boolean): Component {
  vm.$el = el
  if (!vm.$options.render) {
    vm.$options.render = createEmptyVNode
    if (process.env.NODE_ENV !== 'production') {
      /* istanbul ignore if */
      if ((vm.$options.template && vm.$options.template.charAt(0) !== '#') || vm.$options.el || el) {
        warn(
          'You are using the runtime-only build of Vue where the template ' +
            'compiler is not available. Either pre-compile the templates into ' +
            'render functions, or use the compiler-included build.',
          vm
        )
      } else {
        warn('Failed to mount component: template or render function not defined.', vm)
      }
    }
  }
  callHook(vm, 'beforeMount')

  let updateComponent
  /* istanbul ignore if */
  if (process.env.NODE_ENV !== 'production' && config.performance && mark) {
    updateComponent = () => {
      const name = vm._name
      const id = vm._uid
      const startTag = `vue-perf-start:${id}`
      const endTag = `vue-perf-end:${id}`

      mark(startTag)
      const vnode = vm._render()
      mark(endTag)
      measure(`vue ${name} render`, startTag, endTag)

      mark(startTag)
      vm._update(vnode, hydrating)
      mark(endTag)
      measure(`vue ${name} patch`, startTag, endTag)
    }
  } else {
    updateComponent = () => {
      // debugger
      vm._update(vm._render(), hydrating)
    }
  }

  // we set this to vm._watcher inside the watcher's constructor
  // since the watcher's initial patch may call $forceUpdate (e.g. inside child
  // component's mounted hook), which relies on vm._watcher being already defined
  new Watcher(
    vm,
    updateComponent,
    noop,
    {
      before() {
        if (vm._isMounted && !vm._isDestroyed) {
          callHook(vm, 'beforeUpdate')
        }
      },
    },
    true /* isRenderWatcher */
  )
  hydrating = false

  // manually mounted instance, call mounted on self
  // mounted is called for render-created child components in its inserted hook
  if (vm.$vnode == null) {
    vm._isMounted = true
    callHook(vm, 'mounted')
  }
  return vm
}
```
