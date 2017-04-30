---
id: 28
title: How does Facebook Single Sign On work in Android?
date: 2014-05-14T00:54:01+00:00
author: pungrue26@gmail.com
layout: post
guid: http://www.joooooooooonhokim.com/?p=28
permalink: /2014/05/14/how-does-facebook-single-sign-on-work-in-android/
categories:
  - Android
tags:
  - Android
  - Facebook
  - SSO
---
A number of apps provide Facebook Single Sign On(&#8220;Log in with Facebook&#8221;) in their authentication flow. I don&#8217;t even need to present examples for this, I think.

The benefits of using Facebook SSO is not restricted to alleviate annoying signing up process for new users, but they can also leverage user&#8217;s social graph to enrich their services. For example, there is a funny Android app that uses pictures of current user&#8217;s &#8220;friends of (their) friends&#8221;, [_&#8220;너말고니친구&#8221;_](https://play.google.com/store/apps/details?id=com.ultracaption.limit).

<!--more-->

So, I wondered how it works under the hood. Facebook SDK for Android is about 28k lines of code.(v 3.8.0), and some people may feel troublesome for digging into it.

&nbsp;

[<img class="aligncenter wp-image-33 size-full" src="http://www.joooooooooonhokim.com/wp-content/uploads/2014/05/스크린샷-2014-05-15-오전-11.00.14.png" alt="스크린샷 2014-05-15 오전 11.00.14" width="1089" height="67" srcset="http://www.joooonho.com/wp-content/uploads/2014/05/스크린샷-2014-05-15-오전-11.00.14.png 1089w, http://www.joooonho.com/wp-content/uploads/2014/05/스크린샷-2014-05-15-오전-11.00.14-300x18.png 300w, http://www.joooonho.com/wp-content/uploads/2014/05/스크린샷-2014-05-15-오전-11.00.14-1024x63.png 1024w" sizes="(max-width: 1089px) 100vw, 1089px" />](http://www.joooooooooonhokim.com/wp-content/uploads/2014/05/스크린샷-2014-05-15-오전-11.00.14.png)

&nbsp;

I dug into it, and found out how it works under the hood. Yes, I know this would be some kind of &#8220;elementary&#8221; for those seasoned developers, but I still think summarizing what I found would be helpful for some people and at least for myself.

So, this short article

  * is intended to solve this question. &#8220;How does Facebook SSO work in Android under the hood?&#8221;
  * is targeted to novice Android developers, who wants to understand the big picture of Facebook SSO system
  * deals with OAuth 2.0 only briefly, instead focuses on Facebook SDK&#8217;s internal implementation

Let&#8217;s get started.

* * *

### Facebook SSO process basically means your app getting a string called &#8220;access token&#8221; from Facebook.

&nbsp;

Facebook SSO process basically means getting a string called &#8220;access token&#8221; from Facebook App(or from Facebook server if app is not installed). What is access token? It is some opaque string looks like this :

> _CAACEdEose0cBABMkYOLbHCMfb9CBZCz5&#8230;_

This string contains information about which app is signing on, who user is, what permissions user granted and so on. Facebook provides [&#8220;Access token debuger&#8221;](https://developers.facebook.com/tools/debug/) in which we can debug access tokens.

&nbsp;

[<img class="aligncenter wp-image-34 size-full" src="http://www.joooooooooonhokim.com/wp-content/uploads/2014/05/스크린샷-2014-05-14-오후-3.51.07.png" alt="스크린샷 2014-05-14 오후 3.51.07" width="1127" height="632" srcset="http://www.joooonho.com/wp-content/uploads/2014/05/스크린샷-2014-05-14-오후-3.51.07.png 1127w, http://www.joooonho.com/wp-content/uploads/2014/05/스크린샷-2014-05-14-오후-3.51.07-300x168.png 300w, http://www.joooonho.com/wp-content/uploads/2014/05/스크린샷-2014-05-14-오후-3.51.07-1024x574.png 1024w" sizes="(max-width: 1127px) 100vw, 1127px" />](http://www.joooooooooonhokim.com/wp-content/uploads/2014/05/스크린샷-2014-05-14-오후-3.51.07.png)

&nbsp;

And this string is used every time you make a Facebook graph API call. Facebook Graph API support OAuth 2.0, and this is how OAuth 2.0 works. ex-Facebook engineer Luke Shepard&#8217;s [this blog article](http://www.sociallipstick.com/?p=239) is really helpful to have the big picture of it. This is short quotes from the article that we need at this point in time.

> <p style="color: #666666;">
>   <em>The <a style="color: #5f5f5f;" href="http://tools.ietf.org/id/draft-ietf-oauth-v2-01.html">OAuth 2.0 spec</a> is divided into two sections:</em>
> </p>
> 
> <p style="color: #666666;">
>   <em>* First, you get an access token</em>
> </p>
> 
> <p style="color: #666666;">
>   <em> * Second, you use the token to access protected resources.</em>
> </p>

For example, as he mentioned in his post, if we append an access token (notice the protocol switches to https), we can see much more information about ourselves

`https://graph.facebook.com/me?fields=id,name,likes&access_token=CAACEdEose0cBALLxYoog59E16z7rv9fV...`

> <pre>"id": "100001075155914",
   "name": "\uae40\uc900\ud638",
   "likes": {
      "data": [
         {
            "category": "Education",
            "category_list": [
               {
                  "id": "151676848220295",
                  "name": "Education"
               }
            ],
            "name": "NHN NEXT",
            "created_time": "2014-05-19T05:15:03+0000",
            "id": "208240115938558"
         },
         {
            "category": "Product/service",
            "category_list": [
               {
                  "id": "187393124625179",
                  "name": "Web Design"
               },
               {
                  "id": "177721448951559",
                  "name": "Workplace & Office"
               }
            ],
            "name": "Facebook Design",
            "created_time": "2014-05-19T00:26:13+0000",
            "id": "75877461389"
         },
         ...
      ]
   }
}
</pre>

&nbsp;

### So, how exactly third party app gets access token from Facebook

&nbsp;

There are many scenarios when user presses &#8220;Login with Facebook&#8221; button on third party app. Facebook app may be installed and user already has logged in, or user just installed the app, but hasn&#8217;t logged in yet. Also user may has granted permission to current third party app, or hasn&#8217;t.

To handle these various scenarios, Facebook SDK  makes 4 handlers and try them in consecutive order. These are

  * [GetTokenAuthHandler](https://github.com/facebook/facebook-android-sdk/blob/master/facebook/src/com/facebook/AuthorizationClient.java#L696-774) &#8211; tries to establish connection with Facebook&#8217;s background service and get token from it.
  * KatanaLoginDialogAuthHandler, [KatanaProxyAuthHandler](https://github.com/facebook/facebook-android-sdk/blob/master/facebook/src/com/facebook/AuthorizationClient.java#L796-871) &#8211; if first try fails, which means Facebook app is not installed, or user hasn&#8217;t signed it yet etc., it tries to launch appropriate Activity. ( _Note_ : <span style="color: #000000;">KatanaLoginDialogAuthHandler </span>is removed from SDK in 3.14 version. See [changes in detail](https://github.com/facebook/facebook-android-sdk/commit/b9d6c3994a7a0b5995809322ae08c9be3887cb38#diff-32))
  * [WebViewAuthHandler](https://github.com/facebook/facebook-android-sdk/blob/master/facebook/src/com/facebook/AuthorizationClient.java#L558-694) &#8211; if all tries fail, WebView is here for last resort.

This is [snippet of codes from SDK](https://github.com/facebook/facebook-android-sdk/blob/master/facebook/src/com/facebook/AuthorizationClient.java#L185-201).

<pre class="brush: java; title: ; notranslate" title="">private List getHandlerTypes(AuthorizationRequest request) {
    ArrayList handlers = new ArrayList();

    final SessionLoginBehavior behavior = request.getLoginBehavior();
    if (behavior.allowsKatanaAuth()) {
        if (!request.isLegacy()) {
            handlers.add(new GetTokenAuthHandler());
            handlers.add(new KatanaLoginDialogAuthHandler());
        }
        handlers.add(new KatanaProxyAuthHandler());
    }

    if (behavior.allowsWebViewAuth()) {
        handlers.add(new WebViewAuthHandler());
    }

    return handlers;
}

...

void tryNextHandler() {
    if (currentHandler != null) {
        logAuthorizationMethodComplete(currentHandler.getNameForLogging(), EVENT_PARAM_METHOD_RESULT_SKIPPED,
                null, null, currentHandler.methodLoggingExtras);
    }

    while (handlersToTry != null &amp;&amp; !handlersToTry.isEmpty()) {
        currentHandler = handlersToTry.remove(0);

        boolean started = tryCurrentHandler();
        Log.i(TAG, "tryNextHandler started : " + started);

        if (started) {
            return;
        }
    }

    if (pendingRequest != null) {
        // We went through all handlers without successfully attempting an auth.
        completeWithFailure();
    }
}
</pre>

&nbsp;

Let&#8217;s start with the first one, _GetTokenAuthHandler_.

1) [GetTokenAuthHandler](https://github.com/facebook/facebook-android-sdk/blob/master/facebook/src/com/facebook/AuthorizationClient.java#L696-774) &#8211; tries to get token from Facebook&#8217;s background service.

<p style="padding-left: 30px;">
  Because it would be the best if it&#8217;s possible to get token without any friction(like pop-up for user&#8217;s consent etc.), first it tries to get token from Facebook&#8217;s background service. If (a) Facebook app is installed, and (b) user already signed in, and (c) user already granted permission that current third party is requesting, everything is prepared and no pop-up would be needed.
</p>

<p style="padding-left: 30px;">
  When Facebook app is installed and started for the first time, it starts running services in background. (<span style="color: #222222;">A </span><a style="color: #258aaf;" href="http://developer.android.com/guide/components/services.html">Service</a><span style="color: #222222;"> is an application component that can perform long-running operations in the background and does not provide a user interface.)</span>
</p>

<p style="padding-left: 30px;">
  <a href="http://www.joooooooooonhokim.com/wp-content/uploads/2014/05/facebook_bg_service_screenshot.png"><img class="aligncenter wp-image-80 size-full" src="http://www.joooooooooonhokim.com/wp-content/uploads/2014/05/facebook_bg_service_screenshot.png" alt="facebook_bg_service_screenshot" width="700" height="631" srcset="http://www.joooonho.com/wp-content/uploads/2014/05/facebook_bg_service_screenshot.png 700w, http://www.joooonho.com/wp-content/uploads/2014/05/facebook_bg_service_screenshot-300x270.png 300w" sizes="(max-width: 700px) 100vw, 700px" /></a>
</p>

<p style="padding-left: 30px;">
  I&#8217;m not sure one of these services in above screenshot handle SSO, or another service is there in Facebook App to handle it. Anyway, this is basically interprocess communication(IPC), namely third party app&#8217;s process communicate with Facebook&#8217;s process to get access token. In Android, this kind of  job can be done using <a href="http://developer.android.com/guide/components/bound-services.html">Bound Service</a>. Bound service is a <span style="color: #222222;">client-server interface that Android framework provide, and third party app can try to bind(connect) to Facebook&#8217;s background service. W</span>hen connection is established, third party app(client) can request to Facebook app(server) to give token. <a href="https://github.com/facebook/facebook-android-sdk/blob/master/facebook/src/com/facebook/internal/PlatformServiceClient.java">Heres&#8217;s code </a>:
</p>

<pre class="brush: java; title: ; notranslate" title="">public boolean start() {
    if (running) {
        return false;
    }

    // Make sure that the service can handle the requested protocol version
    int availableVersion = NativeProtocol.getLatestAvailableProtocolVersionForService(context, protocolVersion);
    if (availableVersion == NativeProtocol.NO_PROTOCOL_AVAILABLE) {
        return false;
    }

    Intent intent = NativeProtocol.createPlatformServiceIntent(context);
    if (intent == null) {
        return false;
    } else {
        running = true;
        context.bindService(intent, this, Context.BIND_AUTO_CREATE);
        return true;
    }
}

...

public void onServiceConnected(ComponentName name, IBinder service) {
    sender = new Messenger(service);
    sendMessage();
}

...

private void sendMessage() {
    Bundle data = new Bundle();
    data.putString(NativeProtocol.EXTRA_APPLICATION_ID, applicationId);

    populateRequestBundle(data);

    Message request = Message.obtain(null, requestMessage);
    request.arg1 = protocolVersion;
    request.setData(data);
    request.replyTo = new Messenger(handler);

    try {
        sender.send(request);
    } catch (RemoteException e) {
        callback(null);
    }
}
</pre>

<p style="padding-left: 30px;">
  If all conditions are met, then Facebook&#8217;s background service send back to your app access token and SSO process is finished.
</p>

2) <span style="color: #000000;">KatanaLoginDialogAuthHandler, <a href="https://github.com/facebook/facebook-android-sdk/blob/master/facebook/src/com/facebook/AuthorizationClient.java#L796-871">KatanaProxyAuthHandler</a> &#8211; </span>If the first try fails, which means user hasn&#8217;t signed it or granted permission yet, it tries to launch appropriate Activity to do the necessary job.

(These two handler&#8217;s work flows are similar, but exist as separate because of Facebook app&#8217;s internal protocol history. So, I&#8217;ll handle these two in one section. _Note_ : <span style="color: #000000;">KatanaLoginDialogAuthHandler </span>is removed from SDK in 3.14 version. See [changes in detail](https://github.com/facebook/facebook-android-sdk/commit/b9d6c3994a7a0b5995809322ae08c9be3887cb38#diff-32))

<p style="padding-left: 30px;">
  If user hasn&#8217;t signed in to Facebook app yet, signing in is the first thing to be done. It is done in Facebook app, and this is important part of OAuth 2.0(Authenticating happens in Facebook&#8217;s territory, so that third party app doesn&#8217;t get user&#8217;s credential). To do this, it launches <em>Platform Activity</em> which belongs to Facebook app. After launching the Activity, signing in and presenting pop-up dialog for user&#8217;s consent are handled by Facebook App. Below log shows <em>FacebookLoginActivity</em> is launched to sign in user to Facebook app.
</p>

<p style="padding-left: 30px;">
  <a href="http://www.joooooooooonhokim.com/wp-content/uploads/2014/05/스크린샷-2014-05-22-오후-12.05.50.png"><img class="aligncenter wp-image-96 size-full" src="http://www.joooooooooonhokim.com/wp-content/uploads/2014/05/스크린샷-2014-05-22-오후-12.05.50.png" alt="스크린샷 2014-05-22 오후 12.05.50" width="925" height="143" srcset="http://www.joooonho.com/wp-content/uploads/2014/05/스크린샷-2014-05-22-오후-12.05.50.png 925w, http://www.joooonho.com/wp-content/uploads/2014/05/스크린샷-2014-05-22-오후-12.05.50-300x46.png 300w" sizes="(max-width: 925px) 100vw, 925px" /></a>
</p>

&nbsp;

<p style="padding-left: 30px;">
  <a href="http://www.joooooooooonhokim.com/wp-content/uploads/2014/05/facebook_loginactivity_dialog_double_screenshot.jpg"><img class="aligncenter wp-image-91 size-full" src="http://www.joooooooooonhokim.com/wp-content/uploads/2014/05/facebook_loginactivity_dialog_double_screenshot.jpg" alt="facebook_loginactivity_dialog_double_screenshot" width="900" height="600" srcset="http://www.joooonho.com/wp-content/uploads/2014/05/facebook_loginactivity_dialog_double_screenshot.jpg 900w, http://www.joooonho.com/wp-content/uploads/2014/05/facebook_loginactivity_dialog_double_screenshot-300x200.jpg 300w" sizes="(max-width: 900px) 100vw, 900px" /></a>
</p>

<p style="padding-left: 30px;">
  After user signing in, and granting permission, access token is delivered to third party app through onActivityResult() callback. More precisely, LoginActivity(com.facebook.LoginActivity) in SDK get onActivityResult()  callback with access token data.
</p>

3)  [WebViewAuthHandler](https://github.com/facebook/facebook-android-sdk/blob/master/facebook/src/com/facebook/AuthorizationClient.java#L558-694) &#8211; if all tries fail, WebView is here for last resort

<p style="padding-left: 30px;">
   WebView is used for last resort in cases when Facebook app is not installed on device. In this case, a WebView is created and prompt login view for user to sign in. This happens in Facebook&#8217;s territory <span class="fnt_e08 N=a:smd.words" lang="en" style="color: #000000;" tabindex="0">like</span><span style="color: #000000;"> </span><span class="fnt_e08 N=a:smd.words" lang="en" style="color: #000000;" tabindex="0">the</span><span style="color: #000000;"> </span><span class="fnt_e08 N=a:smd.words" lang="en" style="color: #000000;" tabindex="0">preceding, so that third party app doesn&#8217;t get user&#8217;s credential. Login page&#8217;s url looks like :</span>
</p>

<p style="padding-left: 30px;">
  <span class="fnt_e08 N=a:smd.words" lang="en" style="color: #000000;" tabindex="0"><code>https://m.facebook.com/dialog/oauth?client_id={client-id}&e2e=...</code> </span>
</p>

<p style="padding-left: 30px;">
  after, signing in and getting permission from user, Facebook server pass access token using <em>redirect_uri</em>, which means Facebook server redirect WebView to some url address which contains access token data. Below url captured from sample app would make it clear.
</p>

<p style="padding-left: 30px;">
  <code>Redirect URL: fbconnect://success#access_token=CAAFDDRlIT2wBABt...&expires_in=5184000</code>
</p>

<p style="padding-left: 30px;">
  WebView captures this url and parse it.  Now access token is ready to use.
</p>

<pre class="brush: java; title: ; notranslate" title="">private class DialogWebViewClient extends WebViewClient {
        @Override
        @SuppressWarnings("deprecation")
        public boolean shouldOverrideUrlLoading(WebView view, String url) {
            Utility.logd(LOG_TAG, "Redirect URL: " + url);
            if (url.startsWith(WebDialog.REDIRECT_URI)) {
                Bundle values = Util.parseUrl(url);
                // HERE! is the part this WebView gets access token.
                String error = values.getString("error");
                ...

                String errorMessage = values.getString("error_msg");
                ...

                if (Utility.isNullOrEmpty(error) && Utility
                        .isNullOrEmpty(errorMessage) && errorCode == FacebookRequestError.INVALID_ERROR_CODE) {
                    sendSuccessToListener(values);
                } else if (error != null && (error.equals("access_denied") ||
                        error.equals("OAuthAccessDeniedException"))) {
                    sendCancelToListener();
                } else {
                    FacebookRequestError requestError = new FacebookRequestError(errorCode, error, errorMessage);
                    sendErrorToListener(new FacebookServiceException(requestError, errorMessage));
                }

                WebDialog.this.dismiss();
                return true;
            } else if (url.startsWith(WebDialog.CANCEL_URI)) {
                sendCancelToListener();
                WebDialog.this.dismiss();
                return true;
            } else if (url.contains(DISPLAY_TOUCH)) {
                return false;
            }
            // launch non-dialog URLs in a full browser
            getContext().startActivity(
                    new Intent(Intent.ACTION_VIEW, Uri.parse(url)));
            return true;
        }

    ...
}
</pre>

* * *

### 

### Conclusion

<p style="font-size: 20px;">
  <p>
    Facebook SDK provides Single Sign On feature, and it eases signing up process for third party app&#8217;s new user and enriches their services.
  </p>
  
  <p>
    It is accomplished via a) communication with Facebook app&#8217;s background service(Bound Service) or b) launching Facebook app&#8217;s Activity or c) loading login page and redirecting using WebView.
  </p>
  
  <p style="text-align: right;">
    <a href="#top">Back to Top</a>
  </p>