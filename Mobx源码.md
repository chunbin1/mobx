# Mobx源码

## autoRun

autoRun创建了一个`Reaction`对象`R`,然后调用`R.schedule_`,把`R`推入到`globalState.pendingReactions`中，然后执行`runReactionsHelper`,从`globalState.pendingReactions`拿出`Reaction`也就是`R`调用其`runReaction_`方法

`runReaction_`方法中调用autoRun传入的`onInvalidate_`方法

```tsx
// onInvalidate_
function (this: Reaction) {
    // 调用Reaction的track方法
    this.track(reactionRunner)
}

function reactionRunner() {
    // view为autoRun的第一个参数，将reaction暴露出去
    view(reaction)
}
```

然后`track`方法调用了`trackDerivedFunction`方法，`R`暴露到`globalState.trackingDerivation`中，然后调用了`a.get()`此时`ObservableValue`会被注入到注入到了`R._newObserving_`中（详见下），将`observing_`注入到`newObserving_`中，然后调用我们传入的方法，然后调用`bindDependencies`，调用`addObserver`将`R`注入到`a.observers_`中。

那这怎么autoRun联系起来呢？看下面的逻辑

## observable.box

以box为例子，box需要手动调用get和set

调用observable.box会返回一个`ObservableValue`对象`A`，这个对象继承自`Atom`，当我们调用`A.get()`的时候`reportObserved(this)`，然后把这个对象加入到`globalState.trackingDerivation`的`newObserving_`中。

当调用`A.set(2)`的时候,会调用到`Atom.prototype.reportChanged`方法,调用`propagateChanged(this)`，然后遍历`A.observers_`，调用了`Reaction.prototype.onBecomeStale_`,调用了`Reaction.prototype.schedule_`方法（回到上面autoRun的逻辑），产生了一个`Reaction`

### observable

以`mobx.observable({value:1})`为例子

1. 先判断类型，走不同的处理，例子中为`plain object`

2. 以object为例

   proxy可以的情况下，

   asDynamicObservableObject -> 
     asObservableObject -> 创建了 ObservableObjectAdministration对象OAdm -> 并注入到target的 $mobx 中
   然后返回OAdm[$mobx].proxy_ =  new Proxy(OAdm,objectProxyTraps)

3. objectProxyTraps
   这个方法把get和set都代理到了ObservableObjectAdministration上的get_、set_方法

### ObservableObjectAdministration对象
#### 创建
这个对象创建的时候会通过extendObservable-> ObservableObjectAdministration.prototype.extend_ -> createAutoAnnotation的extend_方法 ->  observableannotation中的extend_方法，调用ObservableObjectAdministration.prototyoe.defineObservableProperty_方法 -> 然后对`{value:1}`创建了一个ObservableValue的值为1，然后保存到ObservableObjectAdministration.prototype.values_中
#### set_
会调用observableValue.prototype.setNewValue_-> Atom.prototype.reportChanged然后就跟上面`observable.box`的逻辑一样了

## mobx-react原理
