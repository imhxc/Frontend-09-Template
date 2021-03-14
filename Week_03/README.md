学习笔记

[在线 DEMO](https://codesandbox.io/s/goujianfenxisizeyunsuande-ast-d03wq)

## 前言
对于我们而言，我们可以正确的去分辨出一个数学表达式，比如下面：

```
1 + 2 * 3 // 是
1 +++ 2 // 不是
```

但是对于程序而言，如果想让程序能够正确识别一个描述自然数进行四则运算的字符串并不简单。

所以，简单来说，我们需要让程序能够识别四则运算的基本元素和构成。

这里我们以简单的四则运算入手，一个四则运算算式必须由**数字、加、减、乘、除、括号**构成。

> 其实让计算机理解四则运算，就像我们学习英语一样，都是在掌握一门新的语言。一个英文句子必须由一个个正确的英文单词构成。  
> 仅仅学会了英语单词也不够，因为只通过简单的单词并不能构成完成的句子，我们还要引入语法单位，如主谓宾、定状补、动名词、形容词等。  

我们把一个算式认为是若干个项通过加减运算连接，而**每个项又是通过若干个因子通过乘除来运算连接，而每一个项又是若干个因子通过乘除运算连接，每一个因子可以是一个数字，也可以是一个由括号括起来的表达式。**

```
表达式 -> 项 + 项 + 项 ...
项 -> 因子 * 因子 / 因子 ...
因子 -> 数字
因子 -> (表达式)
```

## 分析步骤和目标
1. 定义四则运算。
2. 词法分析，把输入的字符串流变成 token。
3. 语法分析，把 token 变成抽象语法树 AST。
4. 解释执行，后续便利 AST，执行得出结果。

所以我们的目标就是抽象出四则运算的规则，然后通过一些手段对一个四则运算的字符串进行分析，最终生成一个 AST 语法树。

## 术语
这篇文章会设计较多的术语和一些词语含义的定义，如果有一个字没有说清楚，就很容导致理解偏差，因为 AST 的构建其实是对现实世界的抽象。

* Expression(E)：表达式。
* MultiplicativeExpression：乘法表达式。
* AdditiveExpression：加法表达式。
* TokenNumber：0～9 的数字组合。
* Operator：+、-、*、/（加减乘除）。
* Whitespace：<SP> 空格。
* LineTerminator：<LF> <CR> 换行符。
* TerminalSymbol：终结符。
* NoneTerminalSymbol：非终结符。
* AST(Abstract Syntax Tree)：抽象语法树。
* 语法分析：构建 AST 语法树的过程，就是语法分析。
* **token：在本文中，除特殊说明外，token 默认为有意义的可读信息，比如 1 + 2 ，除了 Whitespace、Line Terminator 外，剩余的 1、+、2 便称为 token。**

## 什么是 LL 算法？
LL 算法是语法分析中的一种算法，出了 LL 语法分析算法，还有 LR 语法分析算法。

**LL 算法就是从左到右扫描，然后从左到右规约的一个缩写。**

## 四则运算的词法定义
提前定义出四则运算的词法和语法规则，才能够让计算机去理解、识别。

在计算机中，我们定义有效的四则运算由如下四种：
1. TokenNumber
2. Operator
3. Whitespace
4. LineTerminator

其中，后两种属于无意义的字符，因为空格和换行都是为了美观和可读性，真正有意义的输入元素里面（有意义的称为 token），是前两者。

> 同时注意，TokenNumber 是支持小数点的。  

根据 [BNF 范式（巴科斯范式）](https://www.zhihu.com/question/27051306)，将四则运算抽象出来，我们可以得到如下的产生式：

```
<Expression> ::=
  <AdditiveExpression><EOF>

<AdditiveExpression> ::=
  <MultiplicativeExpression>
  |<AdditiveExpression><+><MultiplicativeExpression>
  |<AdditiveExpression><-><MultiplicativeExpression>

<MultiplicativeExpression> ::=
  <Number>
  |<MultiplicativeExpression><*><Number>
  |<MultiplicativeExpression></><Number>
```

### TerminalSymbol

我们使用 `[]` 将 Additive Expression、MultiplicativeExpression 中的终结符找出来。

```
<AdditiveExpression> ::=
  <MultiplicativeExpression>
  |<AdditiveExpression>[<+>]<MultiplicativeExpression>
  |<AdditiveExpression>[<->]<MultiplicativeExpression>

<MultiplicativeExpression> ::=
  [<Number>]
  |<MultiplicativeExpression>[<*><Number>]   
  |<MultiplicativeExpression>[</><Number>]
```



这里可能看一下比较懵逼，其实这里是**对现实世界中的四则运算的一个抽象的过程**，这里简单的将抽象过程推导一下：

```
// 我们先来定义什么是 AdditiveExpression、MultiplicativeExpression

AdditiveExpression: 1 + 2
MultiplicativeExpression: 3 * 4

// eg.
1 + 2 -> (1 * 1) + (2 * 1)

// 将上面的括号部分抽象出来
AdditiveExpression = MultiplicativeExpression + MultiplicativeExpression
```

> 不过这里需要指出，这里的抽象，只是为了方面我们后面的递归处理。  

## 词法分析（LL）
词法分析有多种方式，这里我们均采用 LL 算法。

词法分析部分，就是根据前面的词法定义，通过某种手段，将字符串变为 token 流，换句话说，我们需要让程序能够逐个字符去分析，提取有效 token。

其中，词法分析的手段一般有两种：状态及和正则表达式。

在具体代码实现之前，我们还是先来分析下产生式，我们拿 AdditiveExpression 来分析：

```
<AdditiveExpression> ::=
  <MultiplicativeExpression>
  |<AdditiveExpression><+><MultiplicativeExpression>
  |<AdditiveExpression><-><MultiplicativeExpression>
```

当程序拿到一个 AdditiveExpression 后，找到的第一个 symbol，可能是一个 MultiplicativeExpression ，也可能是一个 AdditiveExpression。

> symbol 即字符，在这里查找 symbol 就是查找表达式。虽然上面的 <AdditiveExpression> 有三种可能，但是综合来看，第一个 symbol 只有两种情况，因为第 2、3 种可能开头的字符都是 <AdditiveExpression>。  

对于第一个字符可能为 MultiplicativeExpression 的情况，结合前面我们的 MultiplicativeExpression 可以得到如下可能：

```
<AdditiveExpression> ::=
  <Number>
  |<MultiplicativeExpression><*><Number>
  |<MultiplicativeExpression></><Number>
  |<AdditiveExpression><+><MultiplicativeExpression>
  |<AdditiveExpression><-><MultiplicativeExpression>
```

> 对于 Number 的情况，是否需要当成乘法表达式去处理，需要结合后面的字符来针对性的处理。  


### 目标

我们在生成 AST 之前，要先通过词法分析，将字符串中的有效 token 抽离出来，就像分析一个英语句子之前，先抽离出每个英文单词。

我们最想想要得到如下的数据结构：

```javascript
const tokens = [
  { type: 'Number', value: '1024' },
  { type: '+', value: '+' },
  { type: 'Number', value: '2' },
  { type: '*', value: '*' },
  { type: 'Number', value: '256' },
  { type: 'EOF' }
];
```

所以，我们朝着这个目标去着手实现。

实现的方式有两种，一种是状态机、另一种是正则表达式（重点介绍）。

### 实现方式一：通过状态机进行词法分析

状态机这里不做过多介绍，重点介绍后面的正则方式。

[DEMO](https://codesandbox.io/s/goujianfenxisizeyunsuande-ast-d03wq?file=/index.html)

```javascript
const token = [];
const numbers = ['1', '2', '3', '4', '5', '6', '7', '8', '9', '0'];
const operators = ['+', '-', '*', '/'];
function start(char) {
  if (numbers.includes(char)) {
    token.push(char);
    return inNumnber;
  }
  if (operators.includes(char)) {
    emmitToken(char, char);
    return start;
  }
  if (char === ' ') {
    return start;
  }
  if (char === '\r' || char === '\n') {
    return start;
  }
}

function inNumber(char) {
  if (numbers.includes(char)) {
    token.push(char);
    return inNumber;
  } else {
    emmitToken('Number', token.join(''));
    token = [];
    return start(char);
  }
}

function emmitToken(type, value) {
  console.log(value);
}
const input = '1024 * 2 * 256';
const state = start;

for (let c of input.split('')) {
  state = start(c);
}
state(Symbol('EOF'));
```

在控制台会依次打印出如下 token 流信息：

```
1024
+
2
*
256
```


### 实现方式二：通过正则表达式进行词法分析

原理就是利用正则表达式，不断的去扫描一个四则运算表达式中的 token，对于符合的 token 的保存下来。

在开始之前，我先介绍一个 JS RegExp 对象的一个方法，即 `exec()` 方法，因为这个方法我们等下会用到，而且该方法也有一些坑。

#### exec 方法

exec: 检索字符串中的指定值，如果找到，返回一个数组，索引为 0 的值为匹配到的文本，**后续的索引对应正则表达式子表达式的位置，其中索引对应的值为子表达式所匹配到的值**。

看个例子就清楚了：
```javascript
// 简单起见，这里只定义了两个子表达式，即指匹配数字和空格
const regexp = /([0-9]+)|([ ]+)/g

const str = '1 + 2';

// 第一次匹配
regexp.exec(str); // ['1', '1', undefined, index: 0]

// 第二次匹配
regexp.exec(str); // [' ', undefined, ' ', index: 1]

// 第三次匹配
regexp.exec(str); // [' ', undefined, ' ', index: 3]

// 第四次匹配
regexp.exec(str); // ['2', '2', undefined, index: 4]

// 第五次匹配
regexp.exec(str); // null
```

可以发现，在返回的数组中：
* 索引 0 的值为当前匹配到的字符；
* 索引 1 的值为正则表达式中第 1 个子表达式所匹配到的值；
* 索引 2 的值为正则表达式中第 2 个字表示所匹配到到的值；

以此类推…。

并且可以发现返回数组的长度等于正则表达式子表达式的数量 + 1。

另外，数组中还会携带 3 个属性：
* index：匹配到的字符在源字符串中的索引位置。
* input：被查找的字符串，在我们上面的例子中，input 的值就是 str。
* groups

> 其实之类还有个比较有意思的地方，就是数组可以添加额外的属性的，并且不会影响数组的长度。因为数组的本质也是对象。  

 **⚠️exec 的坑：**
 
这里需要特别注意的是是，如果你把正则表达式对象定义在了全局下，exec 方法会变得有一些怪异。具体表现在如果一个A 字符串没有匹配结束，那么当你重新调用 exec 去匹配 B 字符串的时候，并不会从 B 字符串的第一个字符串去开始查找。

其原因是因为正则对象会在自身上挂载一个 lastIndex 属性，这个属性只有当在返回 null 的时候才会重置为 0。

#### 词法分析实现

回到我们的词法分析上来，我们需要先定义一些初始变量：

```javascript
 const ADDITIVE = "+";
 const MINUS = "-";
 const MULTIPLICATIVE = "*";
 const DIVISION = "/";
 const NUMBER = "Number";
 const WHITESPACE = "Whitespace";
 const LINE_TERMINATOR = "LineTerminator";

let source = [];

// 定义正则表达式，其实就是对四则运算规则的一个抽象匹配，通过"|"去匹配各种可能的 token
const regexp = /([0-9\.]+)|([ \t]+)|([\r\n]+)|(\+)|(\-)|(\*)|(\/)/g;

// 定义 token
const dictionary = [
  NUMBER,
  WHITESPACE,
  LINE_TERMINATOR,
  ADDITIVE,
  MINUS,
  MULTIPLICATIVE,
  DIVISION
];

```

**这里需要注意，dictionary 数组中的每一项的类型要和正则表达式中的子表达式的位置一一对象，比如 NUMBER 就必须要子表达式 ([0-9\.]+) 的位置对应。**之所以要求一一对应，是因为后面我们需要通过索引去超找类型。

不断的去扫描表达式，找出 token：

```javascript
function * tokenize(source) {
  let result = null;
  let lastIndex = 0;
  while (true) {
    lastIndex = regexp.lastIndex;
    result = regexp.exec(source);
    if (!result) {
      break;
    }
    if (regexp.lastIndex - lastIndex > result[0].length) {
      alert('发生错误，可能有无法识别的字符');
      break;
    }
    let token = {
      type: null,
      value: null
    }

    // 根据前面 exec 方法的介绍
    // 所以这里我们从 1 开始，去查找他到底匹配到了哪种输入元素
    // 另外，由于前面提到的 exec 返回的数据结构的特性，
    // 加上 dictionary 的每一项又和正则表达式子表达式一一对应，
    // 所以我们才可以通过索引的方式拿到当前匹配到的字符的类型。
    for (let i = 1; i <= dictionary.length; i++) {
      if (result[i]) {
        token.type = dictionary[i - 1];
      }
    }
    token.value = result[0]; // 将匹配到的字符存入
    yield token; // 返回 token
  }
  yield { type: 'EOF' }
}

function run() {
  // 利用了 generator 的可迭代特性
  for (let token of tokenize('10 + 10 * 25')) {
  console.log(token);
  // 忽略无效的 token，存入 source 输入，供后续语法分析使用
    if (token.type !== WHITESPACE && token.type !== LINE_TERMINATOR) {
      source.push(token);
    }
  }
}
// 调用
run();
```

在控制台中输出：
![](https://s3.ax1x.com/2021/03/14/60f6D1.png)

## 语法分析
词法分析相当于单词提取，一个完整的英文句子，我们还要有分析出主谓宾、定状补等语法单位，这是我们作为一个人在理解句子的过程。

换到计算机身上，也是如此。

所以，我们要在词法分析的基础上进行语法分析。

根据前面我们定义的产生式，我们需要分别对 AdditiveExpression 和 MultiplicativeExpression 定义方法，用来处理每个表达式的各种情况。

换言之，我们需要定义一个 AdditiveExpression 和 MultiplicativeExpression 函数：


### MultiplicativeExpression 方法

回顾下我们前面定义的 MultiplicativeExpression：

```
<MultiplicativeExpression> ::=
  <Number>
  |<MultiplicativeExpression><*><Number>
  |<MultiplicativeExpression></><Number>
```

可以发现，MultiplicativeExpression 接收的 symbol 类型有 Number、MultiplicativeExpression、*、/ 四种。

综合来看，我们的 MultiplicativeExpression 需要处理三种情况：
1. type 为 Number 的情况。
2. type 为 MultiplicativeExpression 且后面是 * 的情况。
3. type 为 MultiplicativeExpression 且后面是 / 的情况。

其中，对于后两种情况，我们需要进行合并，**产生一个新的非终结符**，因为当一个数字后有 * 或者 / 的时候，说明该表达式并未结束。

```javascript
// 接收前面通过词法分析处理后的对象
function MultiplicativeExpression(source) {
  // 一个 Number 我们可以形成一个 MultiplicativeExpression
  if (source[0].type === NUMBER) {
    // 生成一个新的节点，即新的 NoneTerminalSymbol
    let node = {
      type: 'MultiplicativeExpression',
      children: [source[0]]
    }
    source[0] = node;
    // 由于表达式后面可能还有 * 或者 / ，所以需要进行递归
    return MultiplicativeExpression(source); // 递归
  }
  // 这里通过索引位置来判断当前节点的后面是否有值，并且等于 * 
  if (source[0].type === 'MultiplicativeExpression' && source[1] && source[1].type === MULTIPLICATIVE) {
    let node = {
      type: 'MultiplicativeExpression',
      operator: MULTIPLICATIVE,
      children: []
    }
    // 这里大家其实可以想象，当一个 Number 变成了一个表达式
    // 比如 2 -> 2 * 1,
    // 那么我们就需要将 2 * 1 合成一个表达式,
    // 所以这里就要讲前三项重新挂载到 children 上,
    // 这样就构成了一个是 tree,
    node.children.push(source.shift());
    node.children.push(source.shift());
    node.children.push(source.shift());
    source.unshift(node); // 将新生成的结构在放回 source
    return MultiplicativeExpression(source); // 再次递归
  }
  // 除法和前面的原理一样
  if (source[0].type === 'MultiplicativeExpression' && source[1] && source[1].type === DIVISION) {
    let node = {
      type: 'MultiplicativeExpression',
      operator: ,
      children: []
    }
    node.children.push(source.shift());
    node.children.push(source.shift());
    node.children.push(source.shift());
    source.unshift(node);
    return MultiplicativeExpression(source);
  }
  // 递归结束
  // 其实也可以发现，递归结束的标识，
  // 其实就以为这所有的 token 都处理成了 MultiplicativeExpression
  if (source[0].type === 'MultiplicativeExpression') {
    return source[0]
  }
  // 正常情况该语句不会执行，可以忽略
  return MultiplicativeExpression(source);
}
```

### AdditiveExpression 方法

> AdditiveExpression 方法中包含了 MultiplicativeExpression。  

AdditiveExpression 原理也差不多。但是在写之前，我们还是再来回顾下 AdditiveExpression 的产生式：

```
// 注意这个是我们前面推导出来的
<AdditiveExpression> ::=
  <Number>
  |<MultiplicativeExpression><*><Number>
  |<MultiplicativeExpression></><Number>
  |<AdditiveExpression><+><MultiplicativeExpression>
  |<AdditiveExpression><-><MultiplicativeExpression>
```

可以发现，第三项（最后两个）的不是 TerminalSymbol，是一个 MultiplicativeExpression，所以我们在使用第三项之前，需要额外的调用 
MultiplicativeExpression(source) 方法，用来把 source 里面的 NoneTerminalSymbol 给处理掉，即下方代码中 👈 标记的地方。

```javascript
function AdditiveExpression(source) {
  if (source[0].type === 'MultiplicativeExpression') {
    let node = {
      type: 'AdditiveExpression',
      children: [source[0]]
    }
    source[0] = node;
    return AdditiveExpression(source);
  }
    if (source[0].type === 'AdditiveExpression' && source[1] && source[1].type === ADDITIVE) {
    let node = {
      type: 'AdditiveExpression',
      operator: ADDITIVE,
      children: []
    }
    node.children.push(source.shift());
    node.children.push(source.shift());
    MultiplicativeExpression(source); // 👈
    node.children.push(source.shift());
    source.unshift(node);
    return AdditiveExpression(source);
  }
  if (source[0].type === 'AdditiveExpression' && source[1] && source[1].type === MINUS) {
    let node = {
      type: 'AdditiveExpression',
      operator: MINUS,
      children: []
    }
    node.children.push(source.shift());
    node.children.push(source.shift());
    MultiplicativeExpression(source);
    node.children.push(source.shift());
    source.unshift(node);
    return AdditiveExpression(source);
  }
  if (source[0].type === 'AdditiveExpression') {
    return source[0];
  }
  MultiplicativeExpression(source);
  return AdditiveExpression(source);
}
```

### 执行并挂载到顶层 Expression

至此，我们基本上算是完成了四则运算的 AST 树的生成，最后我们再稍微的前进一步，将前面生成的结构进一步挂载到顶级 Experssion 节点上并调用执行。

```javascript
  function Expression(token) {
    if (source[0].type === 'AdditiveExpression' && source[1] && source[1].type === 'EOF') {
      let node = {
        type: 'Expression',
        children: [source.shift(), source.shift()]
      }
      source.shift(node);
      return node;
    }
    AdditiveExpression(source);
    return Expression(source);
  }
```

## 总结
1. 从现实世界中将四则运算规则抽象出来，词法、语法分析就是现实世界的抽象。
2. 抽象出产生式后，根据产生式去分析、思考各种边界情况。
3. 将四则运算简化为加法表达式和乘法表达式的组合，如将 Number 类型的 token 转化为 MultiplicativeExpression。
4. 了解 TerminalSymbol 和 NoneTerminalSymbol，如果是 NoneTerminalSymbol 的时候，继续调用处理成 MultiplicativeExpression。
5. 借用正则，找出有效 token、剔除无效 token。
6. 善用递归思想。
7. JavaScript RegExp 对象中的 exec 方法的使用和特性。
