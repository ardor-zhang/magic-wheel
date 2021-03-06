@[toc]
# 手写轮子系列三——手写模板引擎

## 1. 前言

为了更好的实现视图与业务逻辑（数据状态）更好的分开，模板引擎随之出现，不论是MVP，MVC 还是 MVVM，其中所有的 V 层（view层）都会采用模板引擎，如前端常见的vue中的 `{{ }}`, 服务端渲染常见的 ejs 中的 `<% %>`，这些都是模板引擎。

那模板引擎是如何实现的呢？数据又是如何渲染到 dom 上的呢？下面让我们来一步一步的解析其中原理。

## 2. 功能需求

### 2.1 `{{ }}` 表达式

十分常见的 mustache 表达式，假设有一状态变量 `name=arrow` ,在 `{{ name }}` 中能够渲染对应的值，同时还能够处理中的 JS 表达式，比如 `{{ name.toUpperCase() }}`。

| 模板                                | 渲染的dom结果         |
| ----------------------------------- | --------------------- |
| `<h1>{{ name }}</h1>`               | `<h1>arrow</h1>`      |
| `<h1>{{ name.toUpperCase() }}</h1>` | `<h1>ARROW</h1>`      |
| `<h1>({{ '[' + name + ']' }})</h1>` | `<h1>([arrow])</h1>` |

### 2.2 forEach 遍历

认识并执行 forEach ，也就是需要识别 js api， 假设有 `arr=['aaa', 'bbb']`, 当有如下code 时：

```ejs
{% arr.forEach(item => { %}
    <li>{{ item }}</li>
{% }) %}
```

对应渲染的 DOM 结果应当为 ：

```html
<li>aaa</li>
<li>bbb</li>
```

### 2.3 if 判断

认识并执行 if 语句， 如存在以下code 时：

```ejs
{% if(isShow) { %} 
	<h1>{{ name }}</h1> 
{% }else { %}
	<h1>isShow is false</h1> 
{% } %}
```

当 `isShow`为 `true`时，对应渲染的 DOM 结果应当为 `<h1>arrow</h1>`，当 `isShow`为 `false`时，对应渲染的 DOM 结果应当为空`	<h1>isShow is false</h1> `。

## 3. 实现思路

要实现模板渲染，思路其实非常的简单，下面来一步一步的解析；

1. 首先，获取到的编写好的模板， 比如一个`.ejs`文件，其实就是一串长长的字符串，这个字符串需要我们传递参数到其中，怎么做呢？这就需要用到一个不太常用的 JS 方法，那就是 `new Function(args, str)` ，通过 `new Function` 可以传入字符串然后创建一个匿名函数， `new Function` 对应的两个参数就能够让我们传入字符串以及对应的参数了。如下：

   ```js
   const str = "let str = ''; with(obj) {str += n;} return str"
   // 这样就构造了一个参数为 obj ， 内容为 str 的匿名函数
   const fn = new Function("obj", str);
   console.log(fn)
   
   // 匿名函数  
   // ƒ anonymous(obj) {
   //   let str = ''; 
   //   with(obj) {str += n;} 
   //   return str
   // }
   
   console.log(fn({ n : 2 }));
   // 2
   ```

2. 获取到的字符串很特别，其中包含了一些特殊标记，比如 `{{ }}`, 如果能够把 `{{ }}` 改写成 `${}` （模板字符串），然后在字符串头尾添加 ` 符号，是不是 JS 引擎就可以认识了。

3. 如何改写呢？显然利用正则表达式可实现。

4. 除了`{{ }}` ，那么 `{% %}` 呢？也是很简单的，因为 `{% %}` 这其中的内容一定是 JS 语句，所以只需要将其解析出来，但是这里千万要注意，如果把 js 语句的前后也添加上 ` 符号的话，那么这些 JS 语句也就会变成了字符串，所以这里需要特别注意，当识别（正则）到 JS 语句时，需要特殊处理，如下代码所示，更好理解；

   ```js
   const str1 = "let str = 1; return str"
   const fn1 = new Function(str1);
   console.log(fn1());
   // 1
   
   // let str = 1; return str 是一个 JS 语句，如果添加了 ``，肯定会识别成字符串
   const str2 = "`let str = 1; return str`"
   const fn2 = new Function(str2);
   console.log(fn2());
   // undefined
   
   // 这里一部分是JS语句，一部分是模板字符串，因此需要做这样的特殊区别
   const str3 = "let str = 1; with(obj) { str += `${count}` } return str;"
   const fn3 = new Function('obj' ,str3);
   console.log(fn3({ count: 2 }));
   // 12
   ```

因此，如果通过正则表达式可以解析对应的数据（数据替换成模板字符串）以及 JS 语句，然后利用 `new Function`将参数和内容生成一个匿名函数，是不是就可以实现了。

## 4. 实现步骤

基于第 3 节的思路，代码也就呼之欲出了，这里写出博主的code。

### 4.1 参数的传递

JS 的解析需要特殊的处理，因为`new Function(arg, str)` 中的第一个参数 arg 只能是一个形参名，不能直接传递参数进入，如`new Function({name: 'arrow'}, str)`是不可执行的。那么参数怎么传递呢？难道所有的参数都写成 arg.xxx 的形式吗？显然是不合理的。

```js
// 这样实现是可以的，但是这需要把需要的参数都改造成 obj..
const str1 = "console.log(obj.m, obj.n)";
const fn1 = new Function('obj', str1);
fn1({m: 1, n: 2});
// 1 2

// 如果能直接写 m, n 就好了
// JS 中提供了一个方法， with(obj), 将 with 块中的作用域指向了 obj
// 也就是说， with(obj) { console.log(m, n) } 中的 m, n 不再指向 window, 而是 obj了
const str2 = "with(obj) { console.log(m, n) }";
const fn2 = new Function('obj', str2);
fn2({m: 1, n: 2});
// 1 2
```

因此，需要将 templateStr 全部置于 with(obj) 下。

### 4.2 `{{ }}` 的解析

```js
// 解析 {{  }}  对应的正则
const reMustache = /\{\{([^}]+)\}\}/g;
// 将 {{ arg }} 直接替换成 ${ arg }
templateStr = templateStr.replace(reMustache,  ($0, $1) => {
  return "${" + $1.trim() + "}";
});
```

### 4.3 JS 语句的解析

```js
// 解析 {% %} 对应的正则
const reJsScript = /\{\%([^%]+)\%\}/g;
// 首先直接获取到对应的js语句
// 然后因为 js 不可以被包含在模板字符串中，因此需要特殊处理，前添加一个 `, 承接上一个 `， 后利用 str+= ` 来承接下面的字符串
templateStr = templateStr.replace(reJsScript, ($0, $1) => {
  return "`\n"+$1.trim()+"\nstr+=`";
});
```

### 4.4 完整的生成函数（generator）

在 4.1 中已经解释过下面添加的代码的缘由了。

```js
const head = "let str = ''\nwith(obj){\nstr += `";
const tail = "`\n}\nreturn str;";
const generatorStr = head + templateStr + tail;

console.log(generatorStr);

const generator =  new Function("obj", generatorStr);
```

### 4.5 完整 code

```js
let templateStr = "{% if(isShow) { %} <b>{{ name }}</b> {% }else{ %} isShow is false {% } %} {% arr.forEach(item => { %} <div>{{ item }}</div> {% }) %}"


// 解析 {{  }}  对应的正则
const reMustache = /\{\{([^}]+)\}\}/g;
// 将 {{ arg }} 直接替换成 ${ arg }
templateStr = templateStr.replace(reMustache,  ($0, $1) => {
  return "${" + $1.trim() + "}";
});

// 解析 {% %} 对应的正则
const reJsScript = /\{\%([^%]+)\%\}/g;
// 首先直接获取到对应的js语句
// 然后因为 js 不可以被包含在模板字符串中，因此需要特殊处理，前添加一个 `, 承接上一个 `， 后利用 str+= ` 来承接下面的字符串
templateStr = templateStr.replace(reJsScript, ($0, $1) => {
  return "`\n"+$1.trim()+"\nstr+=`";
});

const head = "let str = ''\nwith(obj){\nstr += `";
const tail = "`\n}\nreturn str;";
const generatorStr = head + templateStr + tail;
console.log(generatorStr);
const generator =  new Function("obj", generatorStr);

const res1 = generator({isShow: true, name: 'arrow', arr: [1, 2]});
console.log(res1);
// <b>arrow</b>   <div>1</div>  <div>2</div>
const res2 = generator({isShow: false, name: 'arrow', arr: [3, 4]});
console.log(res2);
// isShow is false   <div>3</div>  <div>4</div>
```

### 4.6 测试

```js
const compiler = require('../index');

it("解析 {{  }}", () => {
  const output = compiler("<h1>{{ name }}</h1>")({ name: "arrow" });
  expect(output).toBe(`<h1>arrow</h1>`);
});

it("{{}} toUpperCase 表达式", () => {
  const output = compiler("<h1>{{ name.toUpperCase() }}</h1>")({ name: "arrow" });
  expect(output).toBe(`<h1>ARROW</h1>`);
});

it("{% %} js  语句", () => {
  const output = compiler("{% arr.forEach(item => { %}<div>{{ item }}</div>{% }) %}")({ arr: [1, 2] });
  expect(output).toBe(`<div>1</div><div>2</div>`);
});
```
<div align='center'>
  <img src='./img/test.png' style='width: 26%' zoom='26%'/>
</div>


## 5. 源码地址

[点击查看源码地址，顺便给个star吧！！](https://github.com/Ardor-Zhang/magic-wheel/tree/main/03-template-engine)