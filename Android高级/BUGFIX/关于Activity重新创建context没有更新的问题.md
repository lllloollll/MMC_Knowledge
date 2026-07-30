> 问题背景：Activity销毁后，重新走onCreate流程，页面中调用了`Glide.with(itemView).load(url).into(itemView)`，引发了崩溃。
> 
> 原因：1、`Glide.with(itemView)`会获取当前`Activity`的`context`，并进行检查，如果当前获取的`context`为空或者不在生命周期内，则引发崩溃。2、而Activity销毁重建的那一刻，获取到的`context`仍然是上一个`Activity`实例的`context`，而该`context`已不再生命周期状态内。3、一般情况下，`Acitivity`重建后`context`也会更新，发生没有更新的情况，很大概率是`Activity.context`被长生命周期对象持有，未得到释放。
> 
> 解决方法：1、调用`itemView.post{Glide.with(itemView).load(url).into(itemView)}`，等待`Activity`获取自己的`context`。
> 
> 2、排查`Activity.context`调用情况，避免长生命周期对象持有，及时释放资源。


