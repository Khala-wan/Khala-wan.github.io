---
title:  "CoreData-Invalid redeclaration of 'XXX'"
date:   2017-05-31 15:13:21
categories: CoreData 
---

现在Swift已经针对CoreData的API进行了优化，终于不用向以前那么繁琐的写几百行代码使用CoreData。现在创建Entitie对应的NSManagedObject类只需要点击Editor-> Create NSManagedObject SubClass...选择你对应的Entitie即可生成文件。



## 问题
那么问题来了，根据如上办法生成的NSManagedObject的子类会报错
>Invalid redeclaration of 'XXX'

## 解决办法
1.在你的xdatamodeld文件中选中你的Entite。在右侧属性栏选择第三个标签

2.修改Class的Codegen属性为：**Manual/None**
效果如下：
<div align="center"><img style="border: 1px solid #dcdcdc" width="300" height="400" src="https://github.com/Khala-wan/Khala-wan.github.io/raw/master/resource/CoreData/0.jpg"/></div>

3.删除之前生成的文件，再次按照之前的套路生成一遍。

4.Done！开始遨游在CoreData的海洋吧！少年。
