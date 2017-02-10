# Zepto 
  
  * ### 代码结构    
    ```
    (function(global, factory){
        if (typeof define === 'function' && define.amd)
            define(function () { return factory(global) })
        else
            factory(global)
    }(this, function(window){
        //需要执行的function
        var zepto = (function(){
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
* ### 核心方法
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
$.extend(target, [source, [source2, ...]])  
 $.extend(true, target, [source, ...])
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
4. $.extend()