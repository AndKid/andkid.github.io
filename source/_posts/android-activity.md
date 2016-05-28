---
title: Android Activity
date: 2015-10-10 12:38:12
tags:
- android
- activity
---
### startActivityForResult
ActivityA调用startActivityForResult方法启动singleTask的ActivityB  
即便ActivityB调用setResult(Context.Result_OK),ActivityA也得不到正确返回值  
(我测试在ActivityB finish()后ActivityA根本就不调用onActivityResult()了， 但是看startActivityForResult注释说 的是result被设置成了 Context.Result_Cancel)
>For example, if the activity you
are launching uses the singleTask launch mode, it will not run in your
task and thus you will immediately receive a cancel result.

对于singleTask/singleTop在已存在实例的情况下启动该实例会调用 onNewIntent()方法。
