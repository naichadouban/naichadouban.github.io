---
title: 从Dart看js中的async和await
date: 2019-02-13
tags: ["Dart","js","async&await"]
ncategories: ["async&await","js"]
---

今天看Dart的时候，看到下面的解释
原文地址：https://juejin.im/post/5b5005866fb9a04fea589561#heading-9

Dart提供了类似ES7中的async await等异步操作，这种异步操作在Flutter开发中会经常遇到，比如网络或其他IO操作，文件选择等都需要用到异步的知识。
async和await往往是成对出现的，如果一个方法中有耗时的操作，你需要将这个方法设置成async，并给其中的耗时操作加上await关键字，如果这个方法有返回值，你需要将返回值塞到Future中并返回，如下代码所示：

```Dart
Future checkVersion() async {
  var version = await lookUpVersion();
  // Do something with version
}
```
下面的代码使用Dart从网络获取数据并打印出来：
```Dart
import 'dart:async';
import 'package:http/http.dart' as http;

Future<String> getNetData() async{
  http.Response res = await http.get("http://www.baidu.com");
  return res.body;
}

main() {
  getNetData().then((str) {
    print(str);
  });
}

```

# 疑问
既然`getNetData`方法中使用了`await`,为什么后面调用的时候还要使用`.then((){})` ?
我觉得是可以这样调用
```
main() {
  var res = getNetData();
  print(res)
}
```
> 大远解释
如果这样的话，会先`print`打印，打印出来为空。自己想打印的没有打印。
因为函数中的`await`，只能保证函数中包含`await`的代码和下它下一句代码之间是顺序执行的。其他的并不会影响到。
如果你在`getNetData`函数中，后面要使用res，`await`就可以保证函数内部等待`http.get`到结果后再执行。