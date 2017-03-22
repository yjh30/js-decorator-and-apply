# JS装饰者模式及其应用

## 背景
> es5字面量对象支持丰富的赋值表达式，而es6类(class)仅支持函数作为value值，使用装饰者模式，能够在保持声明式语法的前提下提供运行代码的能力

**示例，下列代码在保持声明式语法的前提下提供了运行代码的能力，该文件代码export(导出)的是一个VueComponent组件构造器，赋值方式更丰富**
        
    import { Webview, Component } from 'miox-vue2x';
    import Layout from '@u51/miox-ui-layout';

    @Component({
        components: {
            layout: Layout,
        },
        render(h) {
            return h('div', '这是一个VueComponent组件构造器');
        },
    })
    export default class Test extends Webview {
        // ...
    }



## 设计特点
- 表达式；
- 对函数求值；
- 函数携带 目标对象(target), 方法名(name), 数据或存取描述符(descriptor)作为入参；
- 被求值函数还可返回一个描述符(decorator descriptor) 来修饰目标对象；

**简单描述**：
> 1. 装饰者应用于class类，返回的是装饰函数的返回值，函数第一参数为target目标对象(类本身)

> 2. 装饰者应用于class类方法，函数入参依次为target,name,descriptor，函数也可以返回一个描述符来修饰目标对象属性

## 用法

- 装饰者由@符号，后紧跟一个函数组成
- 装饰者是一个函数表达式，可以像一个工厂函数一样传递附加参数，工厂函数返回一个新的装饰函数
- 既可用来装饰class类，也可用于装饰class类的方法

> **注意：** 装饰者作用于class类，装饰者函数的第一个参数是class类本身(constructor)，
  而作用于class类的方法时，装饰者函数的第一个参数是class类构造器的原型对象prototype

## 简单示例
    function enumerable(target, name, descripter) {
        // descripter.enumerable = true;
        return {
            enumerable: true
        };
    }

    function annotation(target) {
        target.annotated = true; // Add a property on target
    }

    @annotation class Test {
        constructor() {
            this.uid = 0;
        }
        @enumerable getUid() {
            return this.uid;
        }
    }
    
    // getUid方法被@annotation装饰以后变成可枚举
    window.console.log(Object.keys(Test.prototype)); // ['getUid']

    // Test类被@annotated装饰以后，增加了一个值为true的annotated属性
    window.console.log(Test.annotated); // true

## 语法糖
    // 摘自使用miox框架项目中的一段代码
    import { Webview, Component, life, prop } from 'miox-vue2x';

    @Component({
        components: {
            layout: Layout
        }
    })
    export default class IndexPage extends Webview {
        @life mounted() {
            // ...
        }
        @prop isValid() {
            return {
                Type: 'Boolean',
                default: false
            }
        }
    }

- @Component es5语法糖

        var IndexPage = Object.create(Webview.prototype);
        IndexPage = Component({
            components: {
                layout: Layout
            }
        })() || IndexPage;

- @life es5语法糖

        var descriptor = Object.getOwnPropertyDescriptor(IndexPage.prototype, 'bar');
        var desc = life(IndexPage.prototype, 'bar', descriptor);
        if (desc) {
            Object.defineProperty(IndexPage.prototype, 'bar', desc);
        }


## @Compoennt实现原理
- 装饰函数

        function Component(options) {
            if (typeof options === 'function') {
                return componentFactory(options);
            }
            return function (Component) {
                return componentFactory(Component, options);
            };
        }

        function componentFactory(Component, options) {
            // do something ...
            // 以上代码 都是整合options的工作

            const superProto = Object.getPrototypeOf(Component.prototype);
            const Super = superProto instanceof Vue
                ? superProto.constructor
                : Vue;
            const outComponent = Super.extend(options);

            for ( let outs in statics ){
                outComponent[outs] = statics[outs];
            }

            return outComponent;
        }

- 代码解析

> 1. @Component装饰者最终返回的是outComponent，是一个VueComponent构造函数(类)；

> 2. componentFactory函数中，Super为什么不直接写成Vue，这是因为使用@Component装饰过的组件都是一个VueComponent组件，
这些组件有可能存在继承关系；Vue是一个顶级的构造器，如：Webview，大家平时写的组件都是使用Vue.extend生成的VueComponent组件；

例如：miox源码中 Webview继承了Vue，如下：

    @Component
    export default Webview extends Vue {
        // ...
    }

为什么如此写法，因为其他组件继承Webview，继承Webview的组件原型对象为Vue的实例，下面条件判断为true

    const superProto = Object.getPrototypeOf(Component.prototype);
    const bl = superProto instanceof Vue; // true


