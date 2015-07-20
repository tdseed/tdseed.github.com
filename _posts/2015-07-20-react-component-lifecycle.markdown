---
layout: post
title:  "react component lifecycle"
date:   2015-07-20 10:27:25
categories: javascript
---

每一个React组件在加载时都有特定的生命周期，在此期间不同的方法会被执行。

组件加载: componentWillMount

componentWillMount()
componentWillMount会在组件render之前执行，并且永远都只执行一次。

由于这个方法始终只执行一次，所以如果在这里定义了setState方法之后，页面永远都只会在加载前更新一次。

组件加载: componentDidMount

componentDidMount()
这个方法会在组件加载完毕之后立即执行。在这个时候之后组件已经生成了对应的DOM结构，可以通过this.getDOMNode()来进行访问。

如果你想和其他JavaScript框架一起使用，可以在这个方法中执行setTimeout, setInterval或者发送AJAX请求等操作(防止异部操作阻塞UI)。

componentDidMount: function() {
  setTimeout(function() {
    this.setState({items: {name: 'test'}})
  }, 1000)
}
组件更新: componentWillReceiveProps

componentWillReceiveProps(object nextProps)
在组件接收到一个新的prop时被执行。这个方法在初始化render时不会被调用。

Use this as an opportunity to react to a prop transition before render() is called by updating the state using this.setState(). (不太懂这句话。。。)

旧的props可以通过this.props来获取。在这个函数内调用this.setState()方法不会增加一次新的render.

componentWillReceiveProps: function(nextProps) {
  this.setState({
    likesIncreasing: nextProps.likeCount > this.props.likeCount
  });
}
组件更新: shouldComponentUpdate

boolean shouldComponentUpdate(object nextProps, object nextState)
返回一个布尔值。在组件接收到新的props或者state时被执行。在初始化时或者使用forceUpdate时不被执行。

可以在你确认不需要更新组件时使用。

shouldComponentUpdate: function(nextProps, nextState) {
  return nextProps.id !== this.props.id;
}
如果shouldComponentUpdate返回false, render()则会在下一个state change之前被完全跳过。(另外componentWillUpdate和 componentDidUpdate也不会被执行)

默认情况下shouldComponentUpdate会返回true.

By default, shouldComponentUpdate always returns true to prevent subtle bugs when state is mutated in place, but if you are careful to always treat state as immutable and to read only from props and state in render() then you can override shouldComponentUpdate with an implementation that compares the old props and state to their replacements.(这句也不太懂。。。)

如果你需要考虑性能，特别是在有上百个组件时，可以使用shouldComponentUpdate来提升应用速度。

组件更新: componentWillUpdate

componentWillUpdate(object nextProps, object nextState)
在组件接收到新的props或者state但还没有render时被执行。在初始化时不会被执行。

一般用在组件发生更新之前。

组件更新: componentDidUpdate

componentDidUpdate(object prevProps, object prevState)
在组件完成更新后立即执行。在初始化时不会被执行。一般会在组件完成更新后被使用。例如清除notification文字等操作。

Unmounting: componentWillUnmount

在组件从DOM unmount后立即执行.

componentDidMount:function(){
    this.inc = setInterval(this.update,500)
},
componentWillUnmount:function(){
    console.log("goodbye cruel world!")
    clearInterval(this.inc)
}
主要用来执行一些必要的清理任务。例如清除setTimeout等函数，或者任意的在componentDidMount创建的DOM元素。

ref: http://facebook.github.io/react/docs/component-specs.html
