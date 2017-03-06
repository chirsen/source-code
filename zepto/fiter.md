# 可借鉴的部分

1. 判断为window对象：
```
obj == obj.window;
``` 

2. document.documentElelement 整个html元素

3. 去重
```
[].filter.call(arr, function(item, index){ return arr.indexOf(item) == index; })
```