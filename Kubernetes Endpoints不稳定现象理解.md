## Kubernetes Endpoints不稳定现象理解

#### 现象

```
之前遇到了Endpoints丢失的情况，知乎大神也对Endpoints不稳定的情况进行了分析，记录下
```

#### 分析

```
类似于SVN，当创建一个版本号为1的文件A，后续陆续创建B C文件，版本号会到3，接下来删除文件A，当前“自身最后依次修改版本”版本号还是1；
回到K8S上，Endpoint Controller watch 版本号的变化，并根据当前的Service情况删除相应的Endpoints，当回到1之后，又重新触发了2 3 版本的事件，导致Endpoints不停的删除重建，好像走到了一个死循环。
```

#### 处理

```
官方已经在v1.8.8+, v1.9.3+, and v1.10.0+等版本解决该bug，解决方法就是对delete事件单独处理，delete时，版本号不变化，依旧是之前的版本号。
```

#### 参考资料

https://zhuanlan.zhihu.com/p/37217575?utm_source=wechat_session&utm_medium=social&utm_oi=668397371420053504&wm=3333_2001&weiboauthoruid=2907485821&wechatShare=1&from=timeline&isappinstalled=0

https://github.com/kubernetes/kubernetes/pull/64209#issuecomment-391403378

https://github.com/kubernetes/kubernetes/issues/58545