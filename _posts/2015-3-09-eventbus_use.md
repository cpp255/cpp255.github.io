---
layout: post
title:  "Eventbus 使用"
date:   2015-03-09 23:38:10
tagline: "Supporting tagline"
tags : Android,Eventbus
published: true
categories: 技术分享
---
# Eventbus
---
Android 开发中，平时用多了 BroadcastReceiver 或者 Intent 进行事件分发，可以试试 [Eventbus](https://github.com/greenrobot/EventBus),进行封装再使用，简洁、快速，效果拔群。   

Eventbus 的使用比较简单，只需要一个事件接收方，一个事件发送方。就是简单的订阅者模式，只要有消息发送，订阅了该消息的对象就可以收到。Eventbus 支持 Activity、Fragment 做为订阅者，只要如下简单的注册即可接收消息，下面的代码我做了一下封装，只要继承这两个类，默认都可以接收到消息：

### 1.消息注册

##### Activity 注册使用方式：

```
import android.os.Bundle;


import de.greenrobot.event.EventBus;


/**
 * ClassName: MessageEventDispatchActivity.java </br>
 * Function: 事件消息注册 Activity </br>
 * @author jdk
 * @date 2014-11-3
 * @Version: 1.0
 */
 
public abstract class MessageEventDispatchActivity extends Activity{
	@Override
	protected void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		// 类似BroadcastReceiver，在开始时要注册一次
		EventBus.getDefault().register(this);
	}

	@Override
	protected void onDestroy() {
		System.gc();
		super.onDestroy();
		// 同样的，结束的时候也要取消订阅
		EventBus.getDefault().unregister(this);
	}

	/**
	 * 需要使用的消息接收方法
	 * 
	 * @param message 这个类为我封装的消息类，最简单的可以替换成任意类(如 String)
	 */
	public abstract void onEvent(EventMessage message);
}

```

##### Fragment 注册使用方式：


```
import de.greenrobot.event.EventBus;
import android.os.Bundle;
import android.support.v4.app.Fragment;

/**
 * ClassName: MessageEventDispatchFragment.java </br>
 * Function: 事件消息注册 Fragment </br>
 * @author jdk
 * @date 2014-11-3
 * @Version: 1.0
 */
public abstract class MessageEventDispatchFragment extends Fragment {
	@Override
	public void onCreate(Bundle savedInstanceState) {
		super.onCreate(savedInstanceState);
		EventBus.getDefault().register(this);
	}

	@Override
	public void onDestroy() {
		super.onDestroy();
		EventBus.getDefault().unregister(this);
	}

	/**
	 * 需要使用的消息接收方法
	 * 
	 * @param message
	 */
	public abstract void onEvent(EventMessage message);
}

```

###2.消息处理

步骤1注册完接收方式后，就要进行消息处理了，只要订阅了，消息发送都会发送给 onEvent() 方法，这个时候就要在这个方法中进行消息过滤或者处理了。

###3.消息发送
前面2个步骤只是做好了收到消息的准备，消息还需要发送，最简单的消息发送如下

```
// message如步骤1中所示，是我定义好的，也可以用其他任意类型。
EventBus.getDefault().post(message);
```

这只是简单是使用。具体的还可以玩出各中花样，我现在的工程中，使用这个来做全局消息分发，首先做一个消息封装，通过这个消息可以携带各类型的信息、协议信息，然后通过 http(volley、async-http)请求、IM 的 socket 通信进行消的封装，从而可以减少代码耦合、提高复用性。等有时间，再把这个封装抽象出来，单独写一次。