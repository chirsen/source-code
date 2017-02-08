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