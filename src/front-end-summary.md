# LazyMan的实现

> LazyMan是典型的流程控制解决方案的列子  
如果不知道什么是LazyMan，请自行google

```javascript
function _LazyMan(_name) {
    var _this = this; // 缓存this
    _this.tasks = []; // 任务初始化为空数组
    _this.tasks.push(function() {
        console.log('Hi! This is ' + _name + '!');
        // 当前匿名函数，没有明确的执行对象，所以函数里面的this指向window，因此访问当前LazyMan对象就要缓存this
        _this.next();
    });
    // setTimeout会开辟另一个执行临时队列，在当前线程（js是单线程）执行完成后才会执行此临时队列的任务
    // 在执行此临时队列时，会按照先前设置时间延迟
    // 如果延迟时间设置为0，就以为着不延迟执行
    // 这样做的意义在于：让当前任务脱离当前执行的线程（有点异步执行的感觉）
    setTimeout(function() {
        _this.next(); // 执行下一个任务
    }, 0);
}

// push函数里面的this和setTimeout函数里面的this都指向全局作用域，所以要缓存当前this指向
_LazyMan.prototype.next = function() {
    // 取出任务队列的任务存入变量
    var _fn = this.tasks.shift();
    // 如果当前任务存在，就执行当前任务（&&左边为真在会去执行右边，有点if语句的感觉）
    _fn && _fn();
}
_LazyMan.prototype.sleep = function(_time) {
    var _this = this;
    _this.tasks.push(function() {
        setTimeout(function() {
            console.log('Wake up after ' + _time);
            _this.next();
        }, _time);
    });
    return _this;
}
_LazyMan.prototype.sleepFirst = function(_time) {
    var _this = this;
    // sleepFirst和sleep原理一样，只是在新增任务时插队了，把当前任务放在了队列的最前面
    _this.tasks.unshift(function() {
        setTimeout(function() {
            console.log('Wake up after ' + _time);
            _this.next();
        }, _time);
    });
    return _this;
}
_LazyMan.prototype.eat = function(_eat) {
    var _this = this;
    _this.tasks.push(function() {
        console.log('Eat ' + _eat);
        _this.next();
    });
    return _this;
}

var LazyMan = function(_name) { // 封装对象
    return new _LazyMan(_name);
}

// 运行测试
LazyMan('hangyangws').eat('apple').sleep(1000).sleepFirst(2000);
// "Wake up after 2000"(2s后输出)
// "Hi! This is hangyangws!"
// "Eat apple" "Wake up after 1000"(1s后输出)
```

# 用JS求出元素的最终的`background-color`，不考虑元素float、absolute情况

> JS获取元素样式方式：  
**widow.getComputedStyle** (标准浏览器中获取CSS文件中设置的样式返回的对象中，驼峰命名和中划线命名的都有，如：`background-color`和`backgroundColor`都有)
**element.style** (获取的是元素行间设置的样式)
**element.currentStyle** (ie低版本)

```javascript
// 获取指定元素的某个CSS样式，兼容IE
var getStyle = function($el, _attr) {
    if(window.getComputedStyle) {
        return window.getComputedStyle($el, null)[_attr]
    }
    if($el.currentStyle) {
        return $el.currentStyle[_attr];
    }
    return $el.style[_attr];
}

var getFinalBackground = function($el) {
    var color = getStyle($el, 'backgroundColor');
    if(color === 'rgba(0, 0, 0, 0)' || color === 'transparent') { // 判断当前元素背景透明，则查询父元素
        return $el.tagName === 'HTML' ?
            'rgb(255, 255, 255)' : // 遇到根节点就返回白色
            arguments.callee($el.parentNode, 'backgroundColor'); // 不是根节点就继续向上找父节点
    } else { // 当前元素背景不透明，则返回颜色值
        return color;
    }
}
```

# 前端优化简述

> 应用优化涉及各个方面，前端优化只是冰山一角  
有人说：“离开系统的性能瓶颈的前端优化都是扯蛋”  
我觉得，我们各司其职，做好前端本职工作就好，想太多有时不一定是好事

### 优化目的

1. 用户角度：页面加载更快、操作响应更快、体验更好
1. 服务端角度：减少请求数、减小请求带宽

### 优化方法

1. 页面优化
    - HTTP请求数
        1. 从设计实现层面简化页面
        1. 合理设置`HTTP`缓存
        1. 资源合并与压缩(example：`CSS Sprites`)
        1. Inline Images（将图片嵌入到页面或style文件）
        1. Lazy Load Images
        1. 避免重复的资源请求
    - 资源优化
        1. 图片格式的选择（非透明大图尽量不用png、PS保存图片为`web格式`且勾选`连续`选项）
    - 资源的无阻塞加载
        1. CSS放在HEAD中
        1. JavaScript置底
        1. Lazy Load Javascript（example：`AMD`）
1. 代码优化
    - DOM操作优化
        1. 减少DOM操作，减少`Reflow和Repaint`
        1. HTML Collection（类数组集合并不是一个静态的结果，表示的仅是特定的查询，每次访问时会重新执行查询需要遍历 HTML Collection时，将它转为数组再访问，以提高性能）
    - JavaScript
        1. 减少作用域链查找（example：缓存全局变量）
        1. 慎用 `with、eval、Function`
        1. 减少闭包的使用（易内存浪费，不仅仅是常驻内存，重要的是，使用不当会造成无效内存的产生）
        1. 直接量、局部变量的使用（对象属性以及数组的访问需要更大的开销）
        1. 减少字符串拼接`+`使用
    - CSS选择符优化
        1. 减少层级，多用class（浏览器解析CSS是从右往左）
    -  HTML结构优化
        1. 使用HTML5 DOCTYPE
        1. 标签闭合、结构分离
        1. Boolean 属性不需要赋值，如果存在则为True（example：`checked、selected`）
        1. 语义化、标签统一整洁
        1. 减少文本和元素混合，并作为另一元素的子元素
        1. 避免使用`<br />、<hr />`

# 跨域之 JSONP

> **同源策略`same-Origin-Policy`**：指浏览器对不同源的脚本或文本的访问方式进行的限制  
**同源**：指两个页面具有相同的**协议**、**主机`也常说域名`**、**端口**三要素缺一不可  
所以在JS代码中访问不同源的数据会提示*跨域警告*，但是浏览器的`<script>`标签可以加载不同源的数据，这样就给我们“可乘之机”：使用**JSONP**跨域  
**JSONP（JSON with Padding）**的基本原理：在HTML页面中创建`<script>`节点，向不同源提交网络请求，实现跨域

- HTML页面中创建`<script>`节点

    ```javascript
    var script = document.createElement('script'); // 创建<script>节点
    script.src = 'http://example.com/getData'; // 添加src属性
    document.getElementsByTagName('HEAD')[0].appendChild(script); // 插入节点到head头
    ```
- 获取数据返回
    - 我们知道`XMLHttpRequest`对象有`onreadystatechange`方法，在请求成功后可以获取`responseText`内容
    - 但是问题来了，使用`JSONP跨域`如何拿到返回的数据，拿到返回的数据后如何立即调用
    - 解决方案是：
        1. 创建一个函数，函数参数为服务端板返回的数据

            ```javascript
            function callBack(responseText) {
                // 操作responseText
            }
            ```
        1. 给script的src属性设置一个参数比如：`http://example.com/getData?name="callBack"`
        1. 服务端接受到GET参数：`name="callBack"`，再得到`callBack`函数名
        1. 服务端以**JS函数调用**的方式返回数据：`callBack({example: 123})`
        1. 浏览器端得到`JS代码`：`callBack({example: 123})`，然后执行代码
        1. 这个时候我们预先设置好的`callBack`函数就被已**回调函数**的方式调用了
- JSONP优点
    - 与`XMLHttpRequest`不同，`JSONP`不受同源策略限制
    - `IE`支持良好
    - 在请求完成后可通过callback的方式传回结果
- JSONP不足
    - 只支持`GET`请求，不支持`POST`请求
    - 服务端需要根据客户端传过来函数名返回数据
    - 只支持网络跨域的请求数据，不能解决不同域的两个页面之间如何进行JS调用的问题

# 跨域之POST

> 虽然`JSONP`可以解决跨域问题，但是`JSONP`是`GET`类型，传输数据大小不及`POST`类型  
如果需要传递大量数据的跨域，就得了解**POST跨域**

- CORS(Cross Origin Resource Sharing，跨域资源共享)

    > 由于跨域访问会有安全问题，所以有了同源策略  
    我们换位思考，被请求数据的服务器如果同意来自*不同源*的请求，是不是意味着“我信任这个源的请求”  
    那么，同源策略这个时候会“失效”，就达成了“跨域”  
    所以只要服务端设置了http响应头`Access-Control-Allow-Origin`应许请求即可
    比如：

    ```php
    // PHP举例
    header("Access-Control-Allow-Origin: *");
    // 或者(推荐方式)
    header("Access-Control-Allow-Origin: http://www.hangyagnws.win");
    ```

    - 优点：同时支持`GET`、`POST`请求
    - 缺点：需要服务端设置`Access-Control-Allow-Origin`

- invisible iframe

- server proxy

- flash proxy

# 用JS实现矩阵的转置

```javascript
function transMatrix(_matrix) { // 矩阵转置函数
    var _l = _matrix.length,
        _index_in,
        _temp;
    while (--_l > 0) {
        _index_in = _l;
        while (_index_in--) {
            // 互换值
            _temp = _matrix[_l][_index_in];
            _matrix[_l][_index_in] = _matrix[_index_in][_l];
            _matrix[_index_in][_l] = _temp;
        }
    }
}

// 测试
var _matrix = [
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9]
];
transMatrix(_matrix);
console.table(_matrix);
// [
//   [1, 4, 7],
//   [2, 5, 8],
//   [3, 6, 9]
// ]
```

# 用JS冒泡排序法
> 排序中的经典方法，用JS实现感觉又不一样

```javascript
function bubbleSort(_arr) {
    var _len = _arr.length - 1,
        _index_out = 0,
        _index_in, // 内层循环索引
        _temp, // 存放临时数据
        _flag; // 判断循环是否要继续
    if (_len > 0) {
        while (_index_out < _len) {
            _flag = false;
            _index_in = 0; // 内层循环每次要从0开始
            while (_index_in < _len - _index_out) {
                if (_arr[_index_in] > _arr[_index_in + 1]) {
                    // 两者值交换
                    _temp = _arr[_index_in];
                    _arr[_index_in] = _arr[_index_in + 1];
                    _arr[_index_in + 1] = _temp;
                    _flag = true;
                }
                _index_in++;
            }
            if (!_flag) {
                // 如果数组已经是顺序的，就不必再循环了
                break;
            }
            _index_out++;
        }
    }
    return _arr;
}
```

# 用JS实现二分查找法
```javascript
/**
 * @param  {[Array]}  _arr         [查找的数组]
 * @param  {[Number]} _wantVal     [查找的值]
 */
function binarySearch(_arr, _wantVal) {
    // 二分查找的前提是数组应该是有序的
    // 初始时，左边从0开始查找，右边从数组的最右边开始查找
    var _left = typeof arguments[2] !== 'undefined' ? arguments[2] : 0,
        _right = typeof arguments[3] !== 'undefined' ? arguments[3] : _arr.length - 1;
    if (_left > _right) {
        // 没有找到相应值
        return null;
    }
    var _middleIndex = Math.floor((_left + _right) / 2);
    if (_arr[_middleIndex] > _wantVal) {
        // 查找的值在左边
        return binarySearch(_arr, _wantVal, _left, _middleIndex - 1);
    }
    if (_arr[_middleIndex] < _wantVal) {
        // 查找的值在右边边
        return binarySearch(_arr, _wantVal, _middleIndex + 1, _right);
    }
    // 找到要查找的值
    return _middleIndex;
}
```

# 用JS实现快排算法

# JS中的浅复制和深复制

### 浅复制和深复制的区别

### 如何实现Object的深复制`递归的方法进行复制/循环的方法`

# CSS下载与DOM树渲

# HTTPS和HTTP有什么区别

# 网络请求头部的相关解释

# POST请求头部

# new一个对象需要注意的

```javascript
function person(_name) {
    this.name = _name;
};
person.name = 'noUseName';
person.prototype.name = 'testNameTwo';
person.prototype.say = function() {
    console.log(this.name);
}

console.log(person.name); // 返回：person。(函数默认有一个name属性（只读），就是函数名)

new person().say(); // 返回：undefined（等价于: “(new person()).say()”）

new person('hangyangws').say(); // 返回：hangyangw

(new person).say(); // 返回：undefined
```

在查找当前实例对象的属性或者方法时候，如果没有找到，会到原型链`__proto__`上找  
观察仔细的同学应该发现了`new person()`和`new person`都可以实例化对象  
区别就是要传参数就必须带小括号  
不带小括号就表示不传参数（浏览器会自动加上小括号）

**提醒**：  
虽然“new”一个对象的时候可以不带小括号  
但是，“new person.speak()”调用方式会报错：“person.speak is not a constructor”  
因为“.”的优先级大于“new”，类似于：“new (person.speak())”

# SSL四次握手和TCP三次握手

[link](https://github.com/jawil/blog/issues/14)

# SSL握手时有对称加密和非对称加密吗

# 从输入url到渲染的整个过程

[link](https://juejin.im/post/5909b21eda2f60005d1ef731)

# 如果父元素的font-size也是采用em表示，那么子元素的font-size怎么计算等?

# bootstrap的基本原理，bootstrap的grid系统

# xss和csrf

# 事件循环

[link](https://segmentfault.com/a/1190000004322358)

实际上，主线程只会做一件事情，就是从消息队列里面取消息、执行消息，再取消息、再执行。当消息队列为空时，就会等待直到消息队列变成非空。而且主线程只有在将当前的消息执行完成后，才会去取下一个消息。这种机制就叫做事件循环机制，取一个消息并执行的过程叫做一次循环

工作线程是生产者，主线程是消费者(只有一个消费者)

工作线程执行异步任务，执行完成后把对应的回调函数封装成一条消息放到消息队列中；主线程不断地从消息队列中取消息并执行，当消息队列空时主线程阻塞，直到消息队列再次非空。

这就是同步和异步的区别。同步可以保证顺序一致，但是容易导致阻塞；异步可以解决阻塞问题，但是会改变顺序性。改变顺序性其实也没有什么大不了的，只不过让程序变得稍微难理解了一些
