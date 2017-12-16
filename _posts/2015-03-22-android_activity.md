---
layout: post
title:  "Android Activity 源码解析"
date:   2015-03-22 16:01:00
tagline: "Supporting tagline"
tags : Android,Activity
published: true
categories: 技术分享
---
# Android Activity 源码解析
---
  Activity 是用得最多的了，平时也只是熟练的使用，知道 launchMode、intent-filte、screenOrientation，然后看下官网的资料，7个步骤，如何切换这些。现在就读一下底层的源码吧。

  类注释中就有很好的说明了，按例，先从类注释开始，主要有这些
  
  ```
  * <p>Topics covered here:
 * <ol>
 * <li><a href="#Fragments">Fragments</a>
 * <li><a href="#ActivityLifecycle">Activity Lifecycle</a>
 * <li><a href="#ConfigurationChanges">Configuration Changes</a>
 * <li><a href="#StartingActivities">Starting Activities and Getting Results</a>
 * <li><a href="#SavingPersistentState">Saving Persistent State</a>
 * <li><a href="#Permissions">Permissions</a>
 * <li><a href="#ProcessLifecycle">Process Lifecycle</a>
 * </ol>
  ```
  
Activity 生命周期的7个方法：

```
 <pre class="prettyprint">
 * public class Activity extends ApplicationContext {
 *     protected void onCreate(Bundle savedInstanceState);
 *
 *     protected void onStart();
 *     
 *     protected void onRestart();
 *
 *     protected void onResume();
 *
 *     protected void onPause();
 *
 *     protected void onStop();
 *
 *     protected void onDestroy();
 * }
 * </pre>
```

捡关键的来说：

```
onResume() <td>Called when the activity will interacting with the user.  At this point your activity is at the top of the activity stack, with user input going to  <p>Always followed by <code>onPause()</code>.</td>
```
onResume()的方法调用后，就会放在 activity stack 的栈顶，用户就可以进行交互。

```
Unless you specify otherwise, a configuration change (such as a change
 * in screen orientation, language, input devices, etc) will cause your
 * current activity to be <em>destroyed</em>, going through the normal activity
 * lifecycle process of {@link #onPause},
 * {@link #onStop}, and {@link #onDestroy} as appropriate.
```
屏幕方向切换、系统语言、输入设备将会引起当前的 activity 调用 destroyed()。

###代码：
类继承结构

```
java.lang.Object   
   ↳	android.content.Context   
 	   ↳	android.content.ContextWrapper   
 	 	   ↳	android.view.ContextThemeWrapper   
 	 	 	   ↳	android.app.Activity
```