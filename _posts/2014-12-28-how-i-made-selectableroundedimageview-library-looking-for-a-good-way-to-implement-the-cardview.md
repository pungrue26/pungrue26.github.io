---
id: 289
title: How I made SelectableRoundedImageView library (looking for a good way to implement the CardView)
date: 2014-12-28T21:34:44+00:00
author: pungrue26@gmail.com
layout: post
guid: http://www.joooooooooonhokim.com/?p=289
permalink: /2014/12/28/how-i-made-selectableroundedimageview-library-looking-for-a-good-way-to-implement-the-cardview/
categories:
  - Android
---
###  {.p1}

### Executive Summary : {.p1}

<p class="p1">
  I wanted to make an app that uses newly introduced <a href="http://developer.android.com/reference/android/support/v7/widget/CardView.html#setPreventCornerOverlap(boolean)">CardView</a> and I wanted to put an ImageView on top of it, as demonstrated in <a href="http://www.google.com/design/spec/components/cards.html#cards-usage">Material Design spec</a> like below.
</p>

<p class="p1">
  <!--more-->
</p>

<p class="p1">
  <a href="http://www.joooooooooonhokim.com/wp-content/uploads/2014/12/API_demo_round_rects_with_wider_background.png"><br /> </a> <a href="http://www.joooooooooonhokim.com/wp-content/uploads/2014/12/cardview_sample_with_wider_background.png"><img class="aligncenter size-full wp-image-311" src="http://www.joooooooooonhokim.com/wp-content/uploads/2014/12/cardview_sample_with_wider_background.png" alt="cardview_sample_with_wider_background" width="1134" height="538" srcset="http://www.joooonho.com/wp-content/uploads/2014/12/cardview_sample_with_wider_background.png 1134w, http://www.joooonho.com/wp-content/uploads/2014/12/cardview_sample_with_wider_background-300x142.png 300w, http://www.joooonho.com/wp-content/uploads/2014/12/cardview_sample_with_wider_background-1024x485.png 1024w" sizes="(max-width: 1134px) 100vw, 1134px" /></a>
</p>

<p class="p1">
  But then I found Android framework&#8217;s ImageView class doesn&#8217;t support rounded corners just for left-top and right-top corners nor other open-source libraries. So I made <a href="https://github.com/pungrue26/SelectableRoundedImageView">one</a>.
</p>

<p class="p1">
  I believe that CardView looks much better when ImageView inside it get rounded only left-top and right-top corners, even though Google&#8217;s Android app, such as Play Newsstand, Play Movie, aren&#8217;t implemented that way. But other some prominent app, such as <a href="https://play.google.com/store/apps/details?id=com.pinterest">Pinterest</a>, has implemented beautiful CardView in Google&#8217;s Material Design way.
</p>

<p class="p1">
  I got basic idea of this implementation from <a href="http://developer.android.com/reference/android/graphics/drawable/PaintDrawable.html">PaintDrawable class</a>, and I could learn valuable lessons about how ImageView.ScaleType works from this work. This article is all about the journey I took from basic idea to full implementation.
</p>

* * *

###  {.p1}

### Getting the Idea {.p1}

<p class="p1">
  (Note: There is <a href="github.com/vinc3m1/RoundedImageView">a good Android open-source project</a>, developed by <a href="http://makeramen.com/">Vince</a>, which supports rounded corners. Unfortunately, you can only apply the same radius on every corner, because it uses <a href="https://github.com/vinc3m1/RoundedImageView/blob/master/roundedimageview/src/main/java/com/makeramen/RoundedDrawable.java">Canvas.drawRoundRect()</a> method which takes only one parameter as radius. So it couldn&#8217;t meet my needs, because I wanted to make it get rounded only top-left and top-right corners.)
</p>

<p class="p1">
  So basically it was something like connecting the dots.
</p>

<p class="p1">
  First, I knew that we can draw rounded rect with BitmapShader and Canvas.<span class="pln">drawRoundRect </span>method which I learned from <a href="http://www.curious-creature.com/2012/12/11/android-recipe-1-image-with-rounded-corners/comment-page-1/">Romain Guy&#8217;s blog article</a>.
</p>

<p class="p1">
  Also, I have seen rounded rect demo in Android framework&#8217;s ApiDemos app that displays rect images with different radii on every corner. You can check below screen in ApiDemos -> Graphics -> RoundRects.
</p>

<p class="p1">
  <img class="aligncenter size-full wp-image-310" src="http://www.joooooooooonhokim.com/wp-content/uploads/2014/12/API_demo_round_rects_with_wider_background.png" alt="API_demo_round_rects_with_wider_background" width="940" height="650" srcset="http://www.joooonho.com/wp-content/uploads/2014/12/API_demo_round_rects_with_wider_background.png 940w, http://www.joooonho.com/wp-content/uploads/2014/12/API_demo_round_rects_with_wider_background-300x207.png 300w" sizes="(max-width: 940px) 100vw, 940px" />
</p>

<p class="p1">
  So, I thought &#8216;Maybe I can make ImageView get rounded on every corner with different radii, if I can successfully combine these two!&#8217;. And it <em>really</em> was. <a href="https://android.googlesource.com/platform/development/+/master/samples/ApiDemos/src/com/example/android/apis/graphics/RoundRects.java"><span class="s1">The ApiDemos app</span></a> used  <a href="http://developer.android.com/reference/android/graphics/drawable/GradientDrawable.html"><span class="s1">GradientDrawable class</span></a> and its instance method <a href="http://developer.android.com/reference/android/graphics/drawable/GradientDrawable.html#setCornerRadii(float%5B%5D)"><span class="s1">setCornerRadii</span></a> to apply different radii on each of the four corners. So, if there is Drawable class which has setCornerRadii method and also supports Paint setting(Because we can apply BitmapShader to Paint), the job would be easily get done. So I searched it, and hooray! <a href="http://developer.android.com/reference/android/graphics/drawable/PaintDrawable.html"><span class="s1">PaintDrawable class</span></a> was just the right one that I looked for.
</p>

<p class="p1">
  I first tried to implement this library using PaintDrawable, but after some digging, I realized I didn&#8217;t even need to use PaintDrawable class. I found that PaintDrawable class extended ShapeDrawable class, which <a href="https://android.googlesource.com/platform/frameworks/base/+/refs/heads/master/graphics/java/android/graphics/drawable/shapes/RoundRectShape.java#78">draws itself using<span class="pln"> Canvas</span><span class="pun">.</span></a><span class="pln"><a href="https://android.googlesource.com/platform/frameworks/base/+/refs/heads/master/graphics/java/android/graphics/drawable/shapes/RoundRectShape.java#78">drawPath method</a> </span>when its shape is <a href="http://developer.android.com/reference/android/graphics/drawable/shapes/RoundRectShape.html">RoundRectShape</a>.
</p>

### How do they use CardView? and What is the best way for using it? {.p1}

<p class="p1">
  So, check below screenshots. I collected only those which put ImageView on top of CardView.
</p>

<p class="p1">
  1) Google Play Newsstand
</p>

<p class="p1">
  <a href="http://www.joooooooooonhokim.com/wp-content/uploads/2014/12/cardview_sample_newsstand_combined.png"><img class="aligncenter size-full wp-image-332" src="http://www.joooooooooonhokim.com/wp-content/uploads/2014/12/cardview_sample_newsstand_combined.png" alt="cardview_sample_newsstand_combined" width="950" height="600" srcset="http://www.joooonho.com/wp-content/uploads/2014/12/cardview_sample_newsstand_combined.png 950w, http://www.joooonho.com/wp-content/uploads/2014/12/cardview_sample_newsstand_combined-300x189.png 300w" sizes="(max-width: 950px) 100vw, 950px" /></a>
</p>

<p class="p1" style="text-align: left;">
  2) Google Map
</p>

<p class="p1" style="text-align: left;">
  <a href="http://www.joooooooooonhokim.com/wp-content/uploads/2014/12/cardview_sample_googlemap.png"><img class="aligncenter size-full wp-image-330" src="http://www.joooooooooonhokim.com/wp-content/uploads/2014/12/cardview_sample_googlemap.png" alt="cardview_sample_googlemap" width="950" height="600" srcset="http://www.joooonho.com/wp-content/uploads/2014/12/cardview_sample_googlemap.png 950w, http://www.joooonho.com/wp-content/uploads/2014/12/cardview_sample_googlemap-300x189.png 300w" sizes="(max-width: 950px) 100vw, 950px" /></a>3) Google Play Movie
</p>

<p class="p1" style="text-align: left;">
  <a href="http://www.joooooooooonhokim.com/wp-content/uploads/2014/12/cardview_sample_play_movie_combined.png"><img class="aligncenter size-full wp-image-336" src="http://www.joooooooooonhokim.com/wp-content/uploads/2014/12/cardview_sample_play_movie_combined.png" alt="cardview_sample_play_movie_combined" width="950" height="600" srcset="http://www.joooonho.com/wp-content/uploads/2014/12/cardview_sample_play_movie_combined.png 950w, http://www.joooonho.com/wp-content/uploads/2014/12/cardview_sample_play_movie_combined-300x189.png 300w" sizes="(max-width: 950px) 100vw, 950px" /></a>
</p>

<p class="p1" style="text-align: left;">
  So, as you can see, ImageViews don&#8217;t get rounded when not only it is placed on top but also placed on left or right. Let&#8217;s see another implementation of CardView.
</p>

<p class="p1" style="text-align: left;">
  4) Pinterest
</p>

<p class="p1" style="text-align: left;">
  <a href="http://www.joooooooooonhokim.com/wp-content/uploads/2014/12/cardview_sample_pinterest.png"><img class="aligncenter size-full wp-image-338" src="http://www.joooooooooonhokim.com/wp-content/uploads/2014/12/cardview_sample_pinterest.png" alt="cardview_sample_pinterest" width="950" height="600" srcset="http://www.joooonho.com/wp-content/uploads/2014/12/cardview_sample_pinterest.png 950w, http://www.joooonho.com/wp-content/uploads/2014/12/cardview_sample_pinterest-300x189.png 300w" sizes="(max-width: 950px) 100vw, 950px" /></a>
</p>

<p class="p1" style="text-align: left;">
  So, Pinterest made ImageView get rounded. What do you think about this? Which way is better for using it?
</p>

<p class="p1" style="text-align: left;">
  Well, there is no &#8220;right&#8221; answer for this matter. <strong>But I believe that CardView looks much better when it is rounded.</strong>
</p>

<p class="p1" style="text-align: left;">
  First, I think it is about identity of CardView, meaning what a widget called CardView should look like, and feel like. Of course, there can be adjustment and variation, but it should have good reason for that, and go along with the original concept of the widget. Material Design Spec guides <a href="http://www.google.com/design/spec/components/cards.html#cards-usage">&#8220;Cards have rounded corners.&#8221; and &#8220;(square corners) is a tile, not a card.&#8221;</a>.
</p>

<p class="p1" style="text-align: left;">
  Second, it looks better when it is rounded in aesthetic regards. Well, I can&#8217;t prove this, but the fact that all the sample images of CardView in Material Design Spec document are rounded could be strong indication for this. Another grounds for this is <a href="http://developer.android.com/reference/android/support/v7/widget/CardView.html#setPreventCornerOverlap(boolean)">CardView clips its children that intersect with rounded corners</a> from Lollipop and after.
</p>

<p class="p1" style="text-align: left;">
  So, I delivered all my thoughts. You can set different radii on every corner of ImageView with my <a href="github.com/pungrue26/SelectableRoundedImageView">SelectableRoundedImageView library</a>. I hope this library can help you guys a little.
</p>

<p class="p1" style="text-align: left;">
  (TO-DO. Arrange how each of ImageView.ScaleTypes works)
</p>

<p class="p1" style="text-align: left;">
  (TO-DO. Compare my approach of implementation with others including path clipping and redrawing)
</p>