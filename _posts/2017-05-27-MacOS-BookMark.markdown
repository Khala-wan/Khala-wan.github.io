---
title:  "MacOS-BookMark解决沙盒化文件读取权限问题"
date:   2017-05-31 12:13:21
categories: MacOS 
---


BookMarkData是Foundation中一个数据类型。它能帮助我们让沙盒化的MacOS App获取沙盒之外的文件信息。

## 沙盒机制
首先让我们先复习下沙盒机制，OSX从10.6系统开始引入沙盒机制,并且规定发布到Mac AppStore的应用必须遵守沙盒机制。这样能防止恶意的App通过系统漏洞攻击系统获取控制权限,保证了OSX系统的安全。沙盒对应用访问的各个系统资源都做了限制。
大致如下：
* 用户文件
* 硬件外设
* 网络
* XPC
* 其他

正如我们创建了一个新的MacOS App的项目后看到的沙盒配置页面。
<div align="center"><img style="border: 1px solid #dcdcdc" width="500" height="300" src="https://github.com/Khala-wan/Khala-wan.github.io/raw/master/resource/BookMark/0.jpg"/></div>

## 遇到的问题
我在开发自己的MacOS APP的时候发现了这样的问题：我在沙盒中配置了用户选择的文件的读写权限。之后利用NSOpenPanel选中某个文件后读取其中的信息。这在每次选择之后都是可以完成的。但是当我把该文件的URL存入DB后。想让APP下一次启动之后再去读取这个文件时就出现了问题。
>报错：Operation cannot be completed

仔细想了一下应该还是权限问题，于是验证：再一次通过NSOpenPanel选择这个文件之后又可以正常读取了。
之后听朋友说这个功能需要用bookmark实现。于是果断尝试bookmark

## BookMark
随后发现了这么几个API：
``` swift
/// Returns bookmark data for the URL, created with specified options and resource values.
public func bookmarkData(options: URL.BookmarkCreationOptions = default, includingResourceValuesForKeys keys: Set<URLResourceKey>? = default, relativeTo url: URL? = default) throws -> Data
```
这个API用来把你从NSOpenPanel获取到的文件URL（file:///User/XXX）通过指定的bookMark创建选项和URLResourceKey生成BookMarkData。我们需要在DB中存入这个BookMarkData。

这个**BookmarkCreationOptions**是什么呢？它是BookMark创建选项的枚举。有如下选项：

* minimalBookmark 
(见名知意，生成一个能被j解析的最小的书签)
* suitableForBookmarkFile
这个枚举用来处理替身文件。关于如何处理替身文件可以看下老谭的这篇文章。[对替身(Alias)文件的操作](http://www.tanhao.me/pieces/1646.html/)
* withSecurityScope
这个就是我们需要的对用户私密文件进行操作的选项了。
* securityScopeAllowOnlyReadAccess
同上，但是只读你懂得。

**URLResourceKey**是适用于文件系统的URL key。一般传nil就行。在API定义中已经赋值defualt。如果想了解更详细，请看Apple文档:[URLResourceKey](https://developer.apple.com/reference/foundation/urlresourcekey)

OK.继续说我们的BookMark。我们通过URL创建了BookMarkData。也存了DB。那么要怎么用呢？来看这个URL的构造方法：
``` swift
/// Initializes a URL that refers to a location specified by resolving bookmark data.
public init?(resolvingBookmarkData data: Data, options: URL.BookmarkResolutionOptions = default, relativeTo url: URL? = default, bookmarkDataIsStale: inout Bool) throws
``` 
通过传入我们之前存DB的BookMarkData和指定的配置。就可以得到我们之前获取到的文件地址。

**relativeTo url** 是我们BookMarkData对应的baseURL。可以传nil使用默认值

**bookmarkDataIsStale** 标识BookMarkData是否过期，如果过期还需要重新创建。

通过BookMark解析出的fileURL有个很好的好处：当用户修改了这个文件的地址。那么我们再次生成的fileURL也会随之改变。这样帮我们省了很多麻烦。

## 使用BookMark生成的URL访问文件
如果你已经尝试了上面的两个API，那么你会发现还是访问不了。因为我们还没用进行最重要的一步：
来着这两个API。
``` swift
public func startAccessingSecurityScopedResource() -> Bool

/// Revokes the access granted to the url by a prior successful call to startAccessingSecurityScopedResource.
public func stopAccessingSecurityScopedResource()
``` 
在我们要开始读取文件的时候要先调用startAccessingSecurityScopedResource()方法。让我们进入安全访问范围。然后就可读取文件了：
``` swift
func exportFileExists(path:URL)->Bool{
    guard !path.absoluteString.contains(".Trash") else {
        return false
    }
    _ = path.startAccessingSecurityScopedResource()
    do{
        let result = try self.contentsOfDirectory(at: path.deletingLastPathComponent(), includingPropertiesForKeys: nil, options: .skipsHiddenFiles).filter({ (url) -> Bool in
        let fileName:String = url.lastPathComponent
        return fileName.contains("txt")
        })
        path.stopAccessingSecurityScopedResource()
        return result.count > 0
    }catch{
        path.stopAccessingSecurityScopedResource()
        return false
    }
}
``` 
>⚠️⚠️⚠️注意：

>1.使用startAccessingSecurityScopedResource()时必须要和stopAccessingSecurityScopedResource()配套使用进行平衡。如果不这样的话会造成泄漏。后果你不想看到的。

>2.虽然这两个方法看上去像retain和release。但是它们并不是操作了引用计数哟。

## 最后
BookMark和URL配合能解决我们在开发沙盒化APP时很多关于资源操作的问题。我在这里抛砖引玉，希望大家多多探索bookMark，并写点东西一起学习~
