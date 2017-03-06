# Zepto 
  
  * ### 代码结构    
    * * *
    ```
    (function(global, factory){
        if (typeof define === 'function' && define.amd)
            define(function () { return factory(global) })
        else
            factory(global)
    }(this, function(window){
        //需要执行的function
        var Zepto = (function(){
            //核心模块；包含许多方法
            ...
            return $;
        })()  
        window.Zepto = Zepto;
        window.$ === undefined && (window.$ = Zepto);
        
        //其他的模块,并且将方法绑定到Zepto上
        ;(function($){
            //event模块
            ...
        })(Zepto);
        
        ;(function($){
            //ajax模块
        })(Zepto);
        ...
        return Zepto; 
    }))
    ```  
    做的事情：  
    1. 通过对define存在与否的的判断，决定是否遵循 异步模块定义 :  
    ` typeof define === 'function' && define.amd `  
    2. 将返回的Zepto对象绑定到window上面：  
    `window.Zepto = Zepto; `  
    3. 将window.$ 指定为返回的Zepto对象(如果 window.$未被占用) :  
     ` window.$ === undefined && (window.$ = Zepto); `   
    4. 返回Zepto对象 :  `return Zepto;  `  
    如果$被占用，可以：  
    ``` 
    Zepto(function($){
        //使用$
    });
    //或者
    (function($){ 
        //可以在里面使用$代表Zepto
        ...    
    })(Zepto); 
    ```   
    
* ### 核心内容
  * * *
1. $() 选择  
    >
    ``` 
    //1. 传入selector("#id"之类), context(上下文，指定在哪个元素下进行查找)
    //例如: $("#test", "body");
    $ = function (selector, context) {
       return zepto.init(selector, context)
    }
    ```  
    之后是进行查找,目标时返回一个Zepto对象，基本的流程：  
    ```
    //2. 查找的判断
    zepto.init = function (selector, context) {
            var dom
            
            //通过传入的selector,如下几种情况：
            /*
            if(为空)
            
            else if(为string)
            
            else if(为function)
            
            else if(为Zepto对象)
            */
            // 将通过selector找到的dom，创建一个Zepto对象
            return zepto.Z(dom, selector)
        }
    ```  
    如果传入的为string,有如下情况:  
    1. 传入的string为html代码段
    ```
    if (selector[0] == '<' && fragmentRE.test(selector))
       dom = zepto.fragment(selector, RegExp.$1, context), selector = null
    ...
    
    //fragment函数
    zepto.fragment = function (html, name, properties) {
        var dom, nodes, container

        // 如果为空标签 ：singleTagRE = /^<(\w+)\s*\/?>(?:<\/\1>|)$/，直接创建
        if (singleTagRE.test(html)) dom = $(document.createElement(RegExp.$1))

        if (!dom) {
            if (html.replace) html = html.replace(tagExpanderRE, "<$1></$2>")
            if (name === undefined) name = fragmentRE.test(html) && RegExp.$1
            if (!(name in containers)) name = '*'
            
            //使用将目标代码段放到父容器中，在全部取出来的方式，得到dom数组
            container = containers[name]
            container.innerHTML = '' + html
            dom = $.each(slice.call(container.childNodes), function () {
                container.removeChild(this)
            })
        }
        
        //如果传入了属性，则进行属性添加( $each() 方法 )
        if (isPlainObject(properties)) {
            nodes = $(dom)
            $.each(properties, function (key, value) {
                if (methodAttributes.indexOf(key) > -1) nodes[key](value)
                else nodes.attr(key, value)
            })
        }

        return dom
    }
    
    ```    
    2. 传入的为string，并且传入了context：  
    ```
    //直接调用$(context).find(selector), find函数为：
    find: function (selector) {
        var result, $this = this
        if (!selector) result = $()
        else if (typeof selector == 'object')
            result = $(selector).filter(function () {
                var node = this
                return emptyArray.some.call($this, function (parent) {
                    return $.contains(parent, node)
                })
            })
        else if (this.length == 1) result = $(zepto.qsa(this[0], selector))
        else result = this.map(function () { return zepto.qsa(this, selector) })
        return result
    }
    ```  
    3. 传入的为选择符("#id", ".class", "tag")
    ```
    //返回dom数组
    zepto.qsa = function (element, selector) {
        var found,
            maybeID = selector[0] == '#',
            maybeClass = !maybeID && selector[0] == '.',
            //去除符号的名字(#id => id)
            nameOnly = maybeID || maybeClass ? selector.slice(1) : selector, 
            isSimple = simpleSelectorRE.test(nameOnly)
        return (element.getElementById && isSimple && maybeID) ? // Safari DocumentFragment doesn't have getElementById
            ((found = element.getElementById(nameOnly)) ? [found] : []) ://为id
            (element.nodeType !== 1 && element.nodeType !== 9 && element.nodeType !== 11) ? [] :
                slice.call(
                    isSimple && !maybeID && element.getElementsByClassName ? 
                        maybeClass ? element.getElementsByClassName(nameOnly) : //为class 
                            element.getElementsByTagName(selector) : // 为标签
                        element.querySelectorAll(selector) // 如果有多个，使用queryAll
                )
    }
    ```
    如果传入的不为string，有如下情况:  
        1. 为函数，则当document加载完成后执行
    ```
    else if (isFunction(selector)) return $(document).ready(selector)
    ```
        2. 如果为Zepto对象，直接返回  
        3. 其后还有为数组和为对象的情况

2. $.camelCase(string) 转换为驼峰命名,  
    $.contains(parent, node) 判断包含关系  
    代码实现：
    ```
    //camelCase
    function (str) { 
        return str.replace(/-+(.)?/g, function (match, chr) { 
            return chr ? chr.toUpperCase() : '' 
        }) 
    }
    //contains
    $.contains = document.documentElement.contains ?
        function (parent, node) {
            return parent !== node && parent.contains(node)
        } :
        function (parent, node) {
            while (node && (node = node.parentNode))
                if (node === parent) return true
            return false
        };
    ```
3. $.each(): 
遍历数组元素或以key-value值对方式遍历对象。回调函数返回 false 时停止遍历。
```
$.each = function (elements, callback) {
    var i, key
    if (likeArray(elements)) {
        for (i = 0; i < elements.length; i++)
            if (callback.call(elements[i], i, elements[i]) === false) return elements
    } else {
        for (key in elements)
            if (callback.call(elements[key], key, elements[key]) === false) return elements
    }
    
    return elements
}
```
4. $.extend():  
$.extend(target, [source, [source2, ...]])  
$.extend(true, target, [source, ...])  
通过源对象，扩展目标对象的属性，源对象属性将覆盖目标对象属性，浅拷贝和深拷贝两种，通过第一参数是否为true指定  
```
$.extend = function (target) {
        //target直接就是参数列表的第一个了（以后要对第一个参数进行判断可借鉴）
        var deep, args = slice.call(arguments, 1)
        if (typeof target == 'boolean') {
            deep = target
            //改变target的值(参数的值可以被更改，相当于var了一个target)  
            //参数列表的第二个成了目标对象,并且将目标对象弹出数组
            target = args.shift()
        }
        //进行源到目标对象的拓展（遍历的时源对象数组）
        args.forEach(function (arg) { extend(target, arg, deep) })  
        //返回目标对象
        return target
    }
    
//进行拓展的函数：(一个递归过程)
function extend(target, source, deep) {
    //遍历源对象的所有属性
    for (key in source)
        //如果为深度复制并且该属性为纯对象（是对象并且不是window）或数组
        if (deep && (isPlainObject(source[key]) || isArray(source[key]))) {
            if (isPlainObject(source[key]) && !isPlainObject(target[key]))
                //源对象的当前属性为纯对象并且目标属性不是对象，给目标对象的该属性赋值为{}
                target[key] = {}
            if (isArray(source[key]) && !isArray(target[key]))
                //源对象的当前属性为数组并且目标属性不是数组，给目标对象的该属性赋值为[]
                target[key] = []
            //递归当前属性
            extend(target[key], source[key], deep)
        }
        //如果当前属性不是引用，给目标对象赋属性
        else if (source[key] !== undefined) target[key] = source[key]
}
```
此处在判断一个对象是不是window时，使用的是`obj == obj.window `  
5. $.fn Zepto对象的一个方法，拥有Zepto的所有可用方法，在代码中也可以看出来：
    ```
    $.fn = {
        map：function(){...},
        slice:function(){...},
        .
        .
        .
    };
    zepto.Z.prototype = Z.prototype = $.fn;
    //在其他模块添加方法时：
    ;(function($){
        $.fn.xxx = function(){...};
        .
        .
        .
    })(Zepto)
    ```  
    向这个对象添加一个方法，所有的Zepto对象就都能访问了，例如：
    ```
    $.fn.empty = function(){
        return this.each(function(){ this.innerHTML = ""; });
    }
    //使用
    $("#id").empty();
    ```  
6. $.grep(items, function(item){...})  
    相当与Array.filter();内部实现也是调用filter。  
    
    $.inArray(el, arr, [fromIndex])，判断el是否存在于arr中，从frfromIndex开始查找。内部实现即为Array.indexOf()方法  
    
    $.isArray(obj) 判断obj是不是数组  
    ```
    Array.isArray ||
    function (object) { return object instanceof Array }
    ```
7. 方法：  
    ******
    add(selector, [context]),将selector添加到选定的元素数组中  
    ```
    //数组拼接，并且去重
    //
    Zepto.fn = {
        add : function(selector, context){
            return $(uniq(this.concat($(selector, context))));
        }
    }
    ```  
    
    ******  
    addClass(name) => self
    addClass(function(index, className){return "xxx"});selectors[index]的className换成XXX  
    ```
    //方法实现
    Zepto.fn = {
        addClass: function (name) {
                if (!name) return this
                return this.each(function (idx) {
                    //对选定元素组的每个元素
                    if (!('className' in this)) return
                    classList = []
                    //返回当前元素的className，如果name传入的是一个函数
                    //执行函数，函数的返回为对应的selectors[index]的clasName
                    var cls = className(this), newName = funcArg(this, name, idx, cls)
                    newName.split(/\s+/g).forEach(function (klass) {
                        if (!$(this).hasClass(klass)) classList.push(klass)
                    }, this)
                    classList.length && className(this, cls + (cls ? " " : "") + classList.join(" "))
                })
            }
        }
        
        //funcArg(this, name, idx, cls)的实现（就是执行name）
        function funcArg(context, arg, idx, payload) {
            return isFunction(arg) ? arg.call(context, idx, payload) : arg
        }
    ```  
    ******  
    after(),before(),append(),prepend();方法名存在一个数组(adjacencyOperators)中，由一个forEach全部添加到$.fn  
    ```
    adjacencyOperators = ['after', 'prepend', 'before', 'append']
    adjacencyOperators.forEach(function (operator, operatorIndex) {
            var inside = operatorIndex % 2 //=> prepend, append

            $.fn[operator] = function () {
                // arguments can be nodes, arrays of nodes, Zepto objects and HTML strings
                var argType, nodes = $.map(arguments, function (arg) {
                    var arr = []
                    argType = type(arg)
                    if (argType == "array") {
                        arg.forEach(function (el) {
                            if (el.nodeType !== undefined) return arr.push(el)
                            else if ($.zepto.isZ(el)) return arr = arr.concat(el.get())
                            arr = arr.concat(zepto.fragment(el))
                        })
                        return arr
                    }
                    return argType == "object" || arg == null ?
                        arg : zepto.fragment(arg)
                }),
                    parent, copyByClone = this.length > 1
                if (nodes.length < 1) return this

                return this.each(function (_, target) {
                    parent = inside ? target : target.parentNode

                    // convert all methods to a "before" operation
                    target = operatorIndex == 0 ? target.nextSibling :
                        operatorIndex == 1 ? target.firstChild :
                            operatorIndex == 2 ? target :
                                null

                    var parentInDocument = $.contains(document.documentElement, parent)

                    nodes.forEach(function (node) {
                        if (copyByClone) node = node.cloneNode(true)
                        else if (!parent) return $(node).remove()

                        parent.insertBefore(node, target)
                        if (parentInDocument) traverseNode(node, function (el) {
                            if (el.nodeName != null && el.nodeName.toUpperCase() === 'SCRIPT' &&
                                (!el.type || el.type === 'text/javascript') && !el.src) {
                                var target = el.ownerDocument ? el.ownerDocument.defaultView : window
                                target['eval'].call(target, el.innerHTML)
                            }
                        })
                    })
                })
            }

            // after    => insertAfter
            // prepend  => prependTo
            // before   => insertBefore
            // append   => appendTo
            $.fn[inside ? operator + 'To' : 'insert' + (operatorIndex ? 'Before' : 'After')] = function (html) {
                $(html)[operator](this)
                return this
            }
        })
    
    ```
    