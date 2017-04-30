---
id: 136
title: Wrong Intent Flag on Several Famous Android Apps.
date: 2014-06-26T22:56:34+00:00
author: pungrue26@gmail.com
layout: post
guid: http://www.joooooooooonhokim.com/?p=136
permalink: /2014/06/26/wrong-intent-flag-on-several-famous-android-apps/
categories:
  - Android
---
I recently noticed [Facebook app for Android](https://play.google.com/store/apps/details?id=com.facebook.katana) (Version 11.0.0.11.23. Latest version uploaded on Jun 21. 2014) misused intent flag on their advertising post. (I extracted [apk file](https://www.dropbox.com/sh/utp96m1m88zflmo/AABBXleJv13U6y8KDWVT4v89a/com.facebook.katana_9.0.0.26.28.apk) and saved it for preservation.)

<!--more-->

[<img class="aligncenter size-full wp-image-172" src="http://www.joooooooooonhokim.com/wp-content/uploads/2014/06/facebook_ad_post_G_pro.png" alt="facebook_ad_post_G_pro" width="1030" height="755" srcset="http://www.joooonho.com/wp-content/uploads/2014/06/facebook_ad_post_G_pro.png 1030w, http://www.joooonho.com/wp-content/uploads/2014/06/facebook_ad_post_G_pro-300x219.png 300w, http://www.joooonho.com/wp-content/uploads/2014/06/facebook_ad_post_G_pro-1024x750.png 1024w" sizes="(max-width: 1030px) 100vw, 1030px" />](http://www.joooooooooonhokim.com/wp-content/uploads/2014/06/facebook_ad_post_G_pro.png)

<p style="text-align: center;">
  <em>Advertising post on Facebook app for Android .</em>
</p>

&nbsp;

When you press &#8220;Install Now&#8221; button on right bottom of post, it takes you _Play Store_ activity in which you can download the advertised app. But the problem here is, if you press &#8216;_Back_&#8216; button, it doesn&#8217;t take you back to Facebook app, but it displays previous activity on _Play Store_ task, if there was _Play Store_ task before you pressed &#8220;Install Now&#8221; button.

My point would be more clear with this illustration :

**Expected :**

[<img class="aligncenter size-full wp-image-193" src="http://www.joooooooooonhokim.com/wp-content/uploads/2014/06/facebook_ad_post_navigation_expected.png" alt="facebook_ad_post_navigation_expected" width="900" height="817" srcset="http://www.joooonho.com/wp-content/uploads/2014/06/facebook_ad_post_navigation_expected.png 900w, http://www.joooonho.com/wp-content/uploads/2014/06/facebook_ad_post_navigation_expected-300x272.png 300w" sizes="(max-width: 900px) 100vw, 900px" />](http://www.joooooooooonhokim.com/wp-content/uploads/2014/06/facebook_ad_post_navigation_expected.png)

&nbsp;

**Current : (In case of _Play Store_ task exist)[<img class="aligncenter size-full wp-image-206" src="http://www.joooooooooonhokim.com/wp-content/uploads/2014/06/facebook_ad_post_navigation_current_flow.png" alt="facebook_ad_post_navigation_current_flow" width="900" height="817" srcset="http://www.joooonho.com/wp-content/uploads/2014/06/facebook_ad_post_navigation_current_flow.png 900w, http://www.joooonho.com/wp-content/uploads/2014/06/facebook_ad_post_navigation_current_flow-300x272.png 300w" sizes="(max-width: 900px) 100vw, 900px" />](http://www.joooooooooonhokim.com/wp-content/uploads/2014/06/facebook_ad_post_navigation_current_flow.png)**

&nbsp;

Also I recorded current flow. (28 seconds)

<div style="text-align: center;">
</div>

&nbsp;

Facebook app&#8217;s this navigation flow has two problems.

First of all, It is against [Android&#8217;s navigation guide](http://developer.android.com/design/patterns/navigation.html#between-apps) about &#8220;Navigation Between Apps&#8221;, which tells navigation flow in this case should be something like above(Expected). So, It can make users confused.

Second, and more importantly, it can produce an unwanted result that loses users. What if there are a lot of activity stack in _Play Store_ task, so pressing back button multiple times doesn&#8217;t bring misguided users back to Facebook app? He or she may feel frustrated, even can feel really annoying. Of course, there are ways to come back to Facebook app. Pressing _Home_ button and relaunching Facebook app will take users where they want, or to open _Recent Apps_ screen(long press _Home_ button on most devices, or right bottom button on Nexus 4 or 5) and choose Facebook also produce the same result. But what if users don&#8217;t know these method? Or what if users don&#8217;t have such willpower to do extra job to come back, because he is too tired or bored. Also it may switch users context, so now user may wants to stay _Play store_ rather go back to Facebook&#8217;s newsfeed. In either cases, this means losing users in Facebook&#8217;s perspective.

&nbsp;

Long story to short, this wrong navigation flow can lead to lose users.

&nbsp;

If so, what causes this wrong flow? It&#8217;s the intent flag named **[FLAG\_ACTIVITY\_NEW_TASK](http://developer.android.com/reference/android/content/Intent.html#FLAG_ACTIVITY_NEW_TASK) **(whose constant value is <span style="color: #222222;">268435456 (0x10000000))</span>. Logcat tells us Facebook&#8217;s &#8220;Install Now&#8221; button uses this flag.

&nbsp;

[<img class="aligncenter size-full wp-image-154" src="http://www.joooooooooonhokim.com/wp-content/uploads/2014/06/facebook_install_now_button_launch_log.png" alt="facebook_install_now_button_launch_log" width="960" height="48" srcset="http://www.joooonho.com/wp-content/uploads/2014/06/facebook_install_now_button_launch_log.png 960w, http://www.joooonho.com/wp-content/uploads/2014/06/facebook_install_now_button_launch_log-300x15.png 300w" sizes="(max-width: 960px) 100vw, 960px" />](http://www.joooooooooonhokim.com/wp-content/uploads/2014/06/facebook_install_now_button_launch_log.png)

<p style="text-align: center;">
  <em>Logcat log on Facebook&#8217;s &#8220;Install Now&#8221; button</em>
</p>

&nbsp;

According to [document](http://developer.android.com/reference/android/content/Intent.html#FLAG_ACTIVITY_NEW_TASK), &#8220;<span style="color: #222222;"><em><strong>When using this flag, if a task is already running for the activity you are now starting, then a new activity will not be started; instead, the current task will simply be brought to the front of the screen with the state it was last in</strong></em>.&#8221;. So, now we understand why &#8220;Install Now&#8221; button works like this. Below illustration would make it more clear what happens when you touch &#8220;Install Now&#8221; button(Also, Google I/O 2012&#8217;s one session, <a href="https://docs.google.com/presentation/d/1BaucBbey81e5qEyq_71hVe4kmd7F2x5nt_TcyuxrgK0/edit#slide=id.p19">&#8220;Navigation in Android&#8221; presentation slide</a> would help to understand the big picture of this.).</span>

&nbsp;

[<img class="aligncenter wp-image-283 size-large" src="http://www.joooooooooonhokim.com/wp-content/uploads/2014/06/Global_Activity_Stack_when_Install_Now_pressed-1024x433.png" alt="Global_Activity_Stack_when_Install_Now_pressed" width="1024" height="433" srcset="http://www.joooonho.com/wp-content/uploads/2014/06/Global_Activity_Stack_when_Install_Now_pressed-1024x433.png 1024w, http://www.joooonho.com/wp-content/uploads/2014/06/Global_Activity_Stack_when_Install_Now_pressed-300x127.png 300w, http://www.joooonho.com/wp-content/uploads/2014/06/Global_Activity_Stack_when_Install_Now_pressed.png 1581w" sizes="(max-width: 1024px) 100vw, 1024px" />](http://www.joooooooooonhokim.com/wp-content/uploads/2014/06/Global_Activity_Stack_when_Install_Now_pressed.png)

&nbsp;

And this flow(launching _Play Store_ activity, and _Back_ button doesn&#8217;t bring users back to calling app) can be found on other famous apps, like KakaoTalk, the No. 1 messenger app in South Korea, and KakaoStory, a photo sharing social network app for KakaoTalk users.

&nbsp;

[<img class="aligncenter size-full wp-image-220" src="http://www.joooooooooonhokim.com/wp-content/uploads/2014/06/KakaoTalk_KakaoStory_croped.png" alt="KakaoTalk_KakaoStory_croped" width="900" height="572" srcset="http://www.joooonho.com/wp-content/uploads/2014/06/KakaoTalk_KakaoStory_croped.png 900w, http://www.joooonho.com/wp-content/uploads/2014/06/KakaoTalk_KakaoStory_croped-300x190.png 300w" sizes="(max-width: 900px) 100vw, 900px" />](http://www.joooooooooonhokim.com/wp-content/uploads/2014/06/KakaoTalk_KakaoStory_croped.png)

<p style="text-align: center;">
  <em>Invitation to KakaoTalk friends(Left) and KakaoStory feed that shows one of user&#8217;s friend has joined some other app.</em>
</p>

&nbsp;

In above screenshot, pressing &#8220;앱으로 연결&#8221; (which means &#8220;Connect to App&#8221;) or touching post on the center of above KakaoStory feed takes you _Play Store_ activity, like Facebook&#8217;s &#8220;Install  Now&#8221; button does(More precisely, there is a middle stage between &#8220;앱으로 연결&#8221; and _Play Store_ activity, but it takes you there anyway). And pressing back button doesn&#8217;t bring users back to calling app, KakaoTalk and KakaoStory respectively, if there was _Play Store_ task before.

&nbsp;

Then, which intent flag should we use to achieve right flow?

&nbsp;

[FLAG\_ACTIVITY\_CLEAR\_WHEN\_TASK_RESET](http://developer.android.com/reference/android/content/Intent.html#FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET) would be the right one in this situation. We need to make the flow that meets 1) pressing Back button at Play Store should bring users back to original app, 2) pressing Home button, and re-launching the app from launcher should launch main activity rather than Play store activity that previously on top of the stack. And FLAG\_ACTIVITY\_CLEAR\_WHEN\_TASK_RESET is the one that meets these two conditions. I wrote a sample app and recorded video (18 seconds).

&nbsp;

<div style="text-align: center;">
</div>

&nbsp;

This is the code snippet that I used in sample app(exactly same with Facebook app&#8217;s &#8220;Install Now&#8221; button except intent flag).

<pre class="brush: java; title: ; notranslate" title="">public void onClickInstallNow(View v) {
    Intent i = new Intent();
    i.setAction(Intent.ACTION_VIEW);
    i.setData(Uri.parse("market://details?id=com.campmobile.noon"));
    i.setComponent(new ComponentName("com.android.vending", "com.google.android.finsky.activities.LaunchUrlHandlerActivity"));
    i.setFlags(Intent.FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET);
} </pre>

&nbsp;

I reported this to Facebook app&#8217;s &#8220;Report a Problem&#8221; corner, and hope to get some response from them. But I really don&#8217;t understand why this navigation flow keeps recurring here and there. Am I missing something?

&nbsp;

&nbsp;

UPDATE : I found that the navigation flow in Facebook for Android app, at 13.0.0.13.14 version (Latest version uploaded on July 22. 2014) has been changed. I don&#8217;t know my report or my post influenced them to change(Cause I didn&#8217;t get any feedback from Facebook team), but anyway it&#8217;s glad to know my thought wasn&#8217;t something wrong.