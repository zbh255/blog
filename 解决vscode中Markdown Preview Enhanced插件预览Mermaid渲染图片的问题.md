## 最近解决了vscode中Markdown Preview Enhanced插件预览Mermaid渲染图片的问题，做个记录。

>  在```Markdown Preview Enhanced``` 插件中预览Mermaid渲染的图片会出现一下的问题：

- 一些字体无法正确解析
- 渲染出来的图像变成了黑色的图像块

> vscode默认的Markdown解析器并没有此问题

```json
​```mermaid
sequenceDiagram
    participant Alice
    participant Bob
    Alice->>John: Hello John, how are you?
    loop Healthcheck
        John->>John: Fight against hypochondria
    end
    Note right of John: Rational thoughts <br/>prevail!
    John-->>Alice: Great!
    John->>Bob: How about you?
    Bob-->>John: Jolly good!
​```
---

​```mermaid
classDiagram
Class01 <|-- AveryLongClass : Cool
Class03 *-- Class04
Class05 o-- Class06
Class07 .. Class08
Class09 --> C2 : Where am i?
Class09 --* C3
Class09 --|> Class07
Class07 : equals()
Class07 : Object[] elementData
Class01 : size()
Class01 : int chimp
Class01 : int gorilla
Class08 <--> C2: Cool label
​```
​```mermaid
gantt
dateFormat  YYYY-MM-DD
title Adding GANTT diagram to mermaid
excludes weekdays 2014-01-10

section A section
Completed task            :done,    des1, 2014-01-06,2014-01-08
Active task               :active,  des2, 2014-01-09, 3d
Future task               :         des3, after des2, 5d
Future task2               :         des4, after des3, 5d
​```
​```mermaid
gantt
       dateFormat  YYYY-MM-DD
       title Adding GANTT diagram functionality to mermaid

       section A section
       Completed task            :done,    des1, 2014-01-06,2014-01-08
       Active task               :active,  des2, 2014-01-09, 3d
       Future task               :         des3, after des2, 5d
       Future task2              :         des4, after des3, 5d

       section Critical tasks
       Completed task in the critical line :crit, done, 2014-01-06,24h
       Implement parser and jison          :crit, done, after des1, 2d
       Create tests for parser             :crit, active, 3d
       Future task in critical line        :crit, 5d
       Create tests for renderer           :2d
       Add to mermaid                      :1d

       section Documentation
       Describe gantt syntax               :active, a1, after des1, 3d
       Add gantt diagram to demo page      :after a1  , 20h
       Add another diagram to demo page    :doc1, after a1  , 48h

       section Last section
       Describe gantt syntax               :after doc1, 3d
       Add gantt diagram to demo page      :20h
       Add another diagram to demo page    :48h
​```
```

![](http://images.xiao-hui.net/gogs_565584326/Files/raw/master/20191209/Screenshot_20191209_160430.png)

---

> 而使用Markdown Preview Enhanced渲染时：

![](http://images.xiao-hui.net/gogs_565584326/Files/raw/master/20191209/Screenshot_20191209_151900.png)

---

> 网上查阅资料之后之后，始终没有找到好的解决方法。通过查看官方的文档，可能是Mermaid默认使用的css的问题，在设置中更换了Mermaid默认使用的css文件，问题解决！
>
> 打开vscode找到首选项->设置，找到MPE的设置选项，修改以下的设置，将default更改为你喜欢的css样式：

---

![](http://images.xiao-hui.net/gogs_565584326/Files/raw/master/20191209/Screenshot_20191209_152230.png)

### 至此，问题解决！

![](http://images.xiao-hui.net/gogs_565584326/Files/raw/master/20191209/Screenshot_20191209_152306.png)