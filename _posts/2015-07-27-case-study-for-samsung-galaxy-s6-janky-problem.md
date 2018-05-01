# 삼성 갤럭시 S6(및 6 엣지)에서 화면 끊김 현상이 발생하는 원인에 대한 사례 연구 (1)


## 요약 :

&#8211; 갤럭시 S6 및 6 엣지(이하 갤6로 줄여 적습니다)에서 화면을 스크롤할 때 끊김 현상이 잘 발생한다는 보고가 제기되었고, GPU 렌더링을 프로파일링 해보면, 실제로 대부분의 앱을 스크롤링을 할 때 거의 모든 프레임을 16ms 안에 그리지 못하고 있는 것이 확인되었습니다.

<p class="p1">
  &#8211; 원인을 찾기 위해 같은 소스 코드의 앱(유다시티 선샤인 앱)을 같은 OS 버전(5.1.1)의 넥서스 5(이하 넥5로 줄여 적습니다)와 갤6 엣지에 설치하고 Systrace와 Tracer 툴을 이용해 분석하였습니다.
</p>

<p class="p1">
  <!--more-->
</p>

<p class="p1">
  &#8211; 확인 결과, 갤럭시 S6에서 GPU 렌더링 그래프가 높게 나오는 것은 dequeueBuffer 작업이 오래 걸리기 때문이었습니다. dequeueBuffer는 화면 상 변경된 내용들을 새로 그리기 위해 SurfaceFlinger에게 버퍼를 달라고 하는 것입니다. dequeueBuffer 작업은 binder 쓰레드에서 수행됩니다.
</p>

<p class="p1">
  &#8211; 다시 dequeueBuffer 작업이 오래 걸리는 이유는, 이 작업을 수행하도록 할당된 binder 쓰레드에 한번에 충분한 cpu 시간이 할당되지 않거나, 작업을 끝마치기 위해 CPU가 재할당되는데 걸리는 시간이 길었기(약 10ms) 때문이었습니다. dequeueBuffer 작업에 걸리는 시간을 cpu 시간만으로 갤6와 넥5를 비교하면 차이가 없었습니다. 그런데 넥5에서는 거의 모든 경우에 한 번에 충분한 cpu시간이 binder 쓰레드에 할당돼서 dequeueBuffer 작업이 바로 끝났습니다. 간혹 2번에 나누어서 cpu시간이 할당되는 경우도 있었지만,  아주 짧은 시간 안에 다시 binder 쓰레드에 cpu가 재할당 돼서 dequeueBuffer 작업이 끝났습니다. 그러나 갤6의 경우에는 dequeueBuffer 작업을 끝내기 어려운 아주 짧은 cpu 시간 만이 binder 쓰레드에 할당되고, 그래서 작업을 끝내지 못한 채로 cpu는 다른 쓰레드에게 할당이 되었습니다. 그리고 약 10ms 뒤에서야 다시 해당 binder 쓰레드에게 cpu시간이 주어지고, 그제서야 dequeueBuffer 작업이 끝나는 현상이 반복되었습니다.
</p>

<p class="p1">
  &#8211; 결국 갤6에서 프레임 rate가 떨어지는 것은 CPU 스케쥴링의 문제인 것으로 추측됩니다.
</p>

<p class="p1">
  &#8211; 이상까지가 제가 확인한 사항들입니다. 추가적으로 확인해야 되는 사항들은, 1) 그러면 구체적으로 갤6와 넥5에서 CPU 스케쥴링이 어떻게 다르게 동작하는지, 또 2) 갤6에서는 glTexImage2D라는 OpenGL ES 명령이 계속 호출되고, 상당히 긴 시간이 소요되는데 (약 5ms)는데 이 명령은 넥5에서는 호출되지 않는 명령이었습니다.  특히 Tracer 툴로 분석을 해보면 이 명령은 실제로 화면을 갱신하는데에는 관련이 없는 것으로 보였습니다.
</p>

### 이야기의 시작

<p class="p1">
  며칠 전, <a href="https://github.com/dalinaum">용욱</a> 님이 <a href="https://www.facebook.com/photo.php?fbid=10153418767838468">페이스북에 갤6와 넥5의 GPU 렌더링 프로파일을 비교하는 스크린샷을 게시</a>했습니다. 구글 플레이 스토어 앱을 넥5와 갤6에서 실행시키면서 GPU  렌더링을 프로파일 한 것이었는데, 훨씬 좋은 하드웨어 사양을 갖고 있는 갤6에서 오히려 프레임 rate가 떨어지는 것으로 나타나고 있었습니다. 넥5에서는 거의 모든 프레임이 16ms 기준 선인 초록색 선 아래에서 그래프가 그려진데 반해서, 갤6에서는 반대로 거의 모든 프레임이 이 기준선을 넘는 것으로 나타나 있었습니다. 그리고 용욱 님 뿐만 아니라 제가 일하고 있는 팀의 갤6를 사용하는 다른 동료도 비슷한 화면 끊김 현상을 경험하였다고 이야기해주었습니다. 원인이 매우 궁금했고, 파고들어 보았습니다.
</p>

<p class="p1">
  우선 화면 상에서는 빨간색 그래프, 특히 주황색 그래프가 갤6의 경우에 월등히 높이 나타나고, 그것이 16ms를 넘기게 되는 주된 원인이었습니다. 그래서 각각의 그래프가 무엇을 의미하는지 찾아보았고, <a href="https://developer.android.com/tools/performance/profile-gpu-rendering/index.html">공식 안드로이드 개발자 사이트</a>에서 확인할 수 있었습니다. 설명을 옮기면,
</p>

  * The **red** section of the bar represents the time spent by Android&#8217;s 2D renderer issuing commands to OpenGL to draw and redraw display lists. The height of this bar is directly proportional to the sum of the time it takes each display list to execute—more display lists equals a taller red bar.
  * The **orange** section of the bar represents the time the CPU is waiting for the GPU to finish its work. If this bar gets tall, it means the app is doing too much work on the GPU.

<p class="note">
  그리고 이어서 아래와 같은 추가 설명이 있었습니다. 특히 GPU에 할당된 일이 너무 많을 때는 주황색 그래프와 빨간색 그래프의 길이가 길게 나타날 수 있다는 추가 설명이 붙어 있었습니다.
</p>

<p class="note">
  <strong>Note:</strong> While this tool is named Profile GPU Rendering, all monitored processes actually occur in the CPU. Rendering happens by submitting commands to the GPU, and the GPU renders the screen asynchronously. In certain situations, the GPU can have too much work to do, and your CPU will have to wait before it can submit new commands. When this happens, you&#8217;ll see spikes in the Process (orange bar) and Execute (red bar) stages, and the command submission will block until more room is made on the GPU command queue.
</p>

<p class="note">
  이 설명들을 통해 색깔 그래프의 대략적인 의미는 알게 되었지만, 정확한 원인 파악을 위해서 <a href="http://developer.android.com/tools/debugging/systrace.html">Systrace</a>를 통해 두 기기를 분석해보았습니다. (참고로 위 추가 설명에서 언급한 CPU 대기 시간과 관련해서는 <a href="https://www.google.com/events/io/io14videos/983b6a9d-39bc-e311-b297-00155d5066d7">2014 Google I/O 세션 중 &#8220;Perf primer : CPU, GPU and your Android game&#8221;</a>  세션에서 조금 더 자세한 설명을 들으실 수 있습니다)
</p>

### Systrace 툴을 이용한 분석

<p class="note">
  먼저 최대한 동일한 조건에서 두 기기를 비교하기 위해 같은 OS 버전(5.1.1)의 넥5와 갤6 엣지(모델명 SM-G925S)를 준비했습니다. 또 JVM 즉, Java 레벨에서 걸리는 시간을 동일하게 통제하기 위해 동일한 소스 코드의 앱을 실험에 사용하기로 했고, <a href="https://github.com/udacity/Sunshine-Version-2">유다시티 선샤인 앱</a>을 실험용 앱으로 선택했습니다. 선샤인 앱은 소스코드가 공개돼 있어서 다른 분들이 테스트해보기에도 좋기 때문입니다. 두 기기에 선샤인 앱을 설치해서 실행시킨 뒤, 위아래로 빠르게 스크롤을 해보니 역시 넥5에서는 부드럽게 화면이 넘어갔지만, 갤6에서는 거의 모든 프레임이 그려지는데 16ms가 넘게 걸렸습니다.
</p>

<p style="text-align: center;">
(넥서스5 선샤인 앱 메인 액티비티 GPU 렌더링)
</p>

<a href="https://i.imgur.com/RJhO3R5"><img src="https://i.imgur.com/RJhO3R5.png" alt="nexus5_gpu_rendering" width="1024" height="576" srcset="https://i.imgur.com/RJhO3R5.png 1024w, https://i.imgur.com/RJhO3R5h.png 300w" sizes="(max-width: 1024px) 100vw, 1024px" /></a>

<p style="text-align: center;">
(갤럭시 S6 선샤인 앱 메인 액티비티 GPU 렌더링)
</p>

<a href="https://i.imgur.com/Hx6dCJ1"><img src="https://i.imgur.com/Hx6dCJ1.png" alt="galaxy_s6_gpu_rendering" width="1024" height="576" srcset="https://i.imgur.com/Hx6dCJ1.png 1024w, i.imgur.com/Hx6dCJ1l.png 300w" sizes="(max-width: 1024px) 100vw, 1024px" /></a>

<p class="note" style="text-align: left;">
  이어서 Systrace 툴을 실행시켜 두 기기를 분석해 보았습니다. 태그 옵션으로는 Graphics, Input, View System, CPU Scheduling을 선택하였습니다. 또 구체적인 OpenGL 명령들을 파악하기 위해 개발자 옵션에서 OpenGL 추적 사용 설정을 Systrace로 활성화하였습니다. 결과 파일을 공유합니다. <a href="https://www.dropbox.com/s/tr0acc8r133b9e4/trace_sunshine_nexus5_with_gl.html?dl=0">넥서스 5 Systrace 결과</a>, <a href="https://www.dropbox.com/s/fne2o4dr9ffn5w3/trace_sunshine_samsung_with_gl.html?dl=0">갤럭시6 Systrace 결과</a>.
</p>

<a href="https://i.imgur.com/fTbcMYk"><img src="https://i.imgur.com/fTbcMYk.png" alt="opengl_systrace2" sizes="(max-width: 1024px) 100vw, 1024px" /></a>

<p class="note" style="text-align: left;">
  Systrace의 결과는 GPU 렌더링 프로필을 확인시켜줌과 동시에, GPU 안에서 일어나는 일들에 대해 좀 더 자세히 알려주었습니다. 참고로, <a href="http://developer.android.com/about/versions/lollipop.html">안드로이드 롤리팝부터는 메인 쓰레드 외에 렌더 쓰레드가 추가</a>되었고, OpenGL 명령들은 이 렌더쓰레드에서 수행됩니다.  렌더쓰레드는 Systrace 결과 화면에서 가장 아래 행에 나타나는데 넥서스 5에서는 대부분의 DrawFrame 과정이 10ms 이내에 끝나는데 비해 갤6에서는 대부분 16ms를 조금 넘는 시간이 걸림을 확인할 수 있었습니다.
</p>

<center>
(넥서스5 선샤인 Systrace)
</center>

<a href="https://i.imgur.com/sc7WdW5"><img src="https://i.imgur.com/sc7WdW5.png" alt="넥서스5 선샤인 Systrace" sizes="(max-width: 1024px) 100vw, 1024px" /></a>

<center>
(갤럭시 S6 선샤인 Systrace)
</center>

<a href="https://i.imgur.com/yClq98K"><img src="https://i.imgur.com/yClq98K.png" alt="갤럭시 S6 선샤인 Systrace" sizes="(max-width: 1024px) 100vw, 1024px" /></a>

<p class="note" style="text-align: left;">
  그런데 보시는 것처럼 DrawFrame을 끝내는데 걸리는 시간이 16ms를 약간 초과하는 정도이기 때문에, 사실 선샤인 앱에서는 육안으로 화면 끊김 현상을 잘 느끼기는 어렵습니다(그리고 갤럭시 S6의 화면 갱신 주기가 얼마인지와도 관련됩니다. <a href="https://youtu.be/Q8m9sHdyXnE?t=16m11s">모든 안드로이드 기기가 60Hz로 화면을 갱신하지는 않으며 몇 Hz로 화면을 갱신하는지는 기기마다 다릅니다</a>). 하지만 그것은 선샤인 앱이 Best practice로 만들어진 앱이고, 아주 간단한 앱이기 때문입니다. 다른 앱에서도 테스트해본 결과, 플레이 스토어 앱이나 심지어 설정 앱 같은 아주 간단한 앱에서조차 빠르게 스크롤을 내리면 프레임 rate가 떨어짐을 확인할 수 있었습니다.
</p>

<h3 class="note" style="text-align: left;">
  범인은 dequeueBuffer 작업
</h3>

위 갤6 스크린샷에서도 나와있는 것처럼, DrawFrame 작업의 대부분의 시간은 eglSwapBuffer를 수행하는데 소요되는데, 그 작업은 다시 대부분의 시간이 dequeueBuffer 작업을 수행하는데 걸림을 알 수 있습니다. 5초 동안 추적한 Systrace의 거의 모든 프레임에서 동일하였습니다. 넥5에서는 <0.5ms이내에 이 작업이 대부분 끝나는데 비해, 갤6에서는 10ms 가까이 시간이 걸렸습니다. 그래서 dequeueBuffer가 어떤 작업인지 찾아보았고, [Google I/O 2012 Butter or Worse 세션](http://commondatastorage.googleapis.com/io2012/presentations/live%20to%20website/109.pdf)에서 잘 설명되어 있었습니다. 이 세션에서의 [Romain Guy의 설명에 따르면](https://youtu.be/Q8m9sHdyXnE?t=13m24s), dequeueBuffer 작업은 화면을 갱신하기 위해 Surfaceflinger에게 버퍼를 요청하는 것입니다(This is we talk to Surfaceflinger, so we have this window compositer, we ask Surfaceflinger to give us a buffer in which we can draw. This is dequeueing a buffer.).

그런데 조금 더 자세히 살펴보니, dequeueBuffer 작업을 수행하는데 걸리는 CPU 시간 자체는 두 기기에서 차이가 나지 않음을 알 수 있었습니다. 갤6에서 dequeueBuffer 작업이 오래 걸리는 것은 이 작업을 처리하는 Binder 쓰레드에 한 번에 충분한 CPU 시간이 할당되지 않고, Binder 쓰레드는 CPU가 다시 할당될 때까지 기다렸다 작업을 이어서 하는데, 이렇게 CPU가 재할당될때까지 약 10ms의 긴 시간이 걸리는 것이 원인이었습니다.

(갤럭시 S6에서 dequeueBuffer 작업이 오래 걸리지만, 실제로 이 작업을 수행하는 Binder 쓰레드에 할당되는 CPU 시간은 짧은 모습)

![alt text](https://i.imgur.com/G41Du1T.png "galaxy_s6_dequeue_binder")

&nbsp;

(처음 Binder 쓰레드에 CPU가 할당되는 시점의 모습. 위 스크린샷의 1765 ms 부분을 확대한 모습)

![alt text](https://i.imgur.com/4lXhUWz.png "galaxy_s6_dequeue_binder1")

&nbsp;

(dequeueBuffer작업을 맡은 Binder 쓰레드에 CPU가 재할당되는 시점의 모습. 위 스크린샷의 1775 ms 부분을 확대한 모습)

![alt text](https://i.imgur.com/6AimHlo.png "galaxy_s6_dequeue_binder2")

&nbsp;

(갤럭시 S6에서 dequeueBuffer 작업이 반복해서 약 10ms 정도씩 소요되는 모습)

![alt text](https://i.imgur.com/G2Y5dLq.png "galaxy_s6_long_dequeue")

&nbsp;

반면 넥5의 경우에는 실제로 dequeueBuffer 작업을 하는데 걸리는 CPU 시간은 비슷하지만, Binder 쓰레드가 한 번에(또는 아주 짧은 간격으로 2번에) dequeueBuffer 작업을 끝냅니다.

(넥서스 5에서 dequeueBuffer 작업을 맡은 Binder 쓰레드에 CPU가 할당되는 모습)

![alt text](https://i.imgur.com/37MR8ur.png "nexus_5_dequeue_binder")

&nbsp;

여기까지 살펴봄으로써 문제의 원인에 조금 더 가까워졌습니다. 추측컨대 CPU 스케쥴링의 최적화 부분에서 두 기기의 성능 차이가 발생하는 듯합니다. 그런데 이 이상은 어떤 도구를 이용해 분석해야 하는지, 어떤 자료를 참고할 수 있는지를 알지 못해 우선 여기까지만 정리해서 공유하려 합니다. 다른 분들의 도움을 통해 원인을 끝까지 밝혀보고 싶습니다. 많은 피드백 바랍니다.

&nbsp;

### 추가로 살펴봐야 할 것들

혹시 다른 추가적인 정보를 얻을 수 있을까하여 [Tracer](http://developer.android.com/tools/help/gltracer.html) 툴을 이용해서도 두 기기를 비교해 보았습니다. ([넥서스 5 gl trace 결과](https://www.dropbox.com/s/boiaazdu0lx4v9q/trace_sunshine_nexus_5.gltrace?dl=0), [갤럭시 S6 gl trace 결과](https://www.dropbox.com/s/kbvbp9b5ebk2asv/trace_sunshine_samsung_galaxy_s6.gltrace?dl=0)).

그 결과 재미있는 점이 또 하나 발견되었는데, 동일한 화면을 GPU가 그리는 데에 사용하는 OpenGL  명령들이 다르고, 순서도 조금 다른 점이 확인되었습니다. 이것은 넥5와 갤6가 서로 다른 GPU를 사용하기 때문인 듯한데([넥5는 퀄컴의 Adreno](http://www.gsmarena.com/lg_nexus_5-5705.php)를, [갤6는 ARM 사의 Mali T760MP8를 사용](http://www.anandtech.com/show/9146/the-samsung-galaxy-s6-and-s6-edge-review/6)한다고 합니다), 그 중 가장 재미있는 점은 갤6에서는 glTexImage2D라는 OpenGL 명령이 반복해서 호출됨에 비해, 넥5에서는 그 명령이 호출되지 않는 점이었습니다. glTexImage2D 명령은 소요되는 시간도 꽤 긴 편인데(1 ~10ms), 더욱 흥미로운 점은 Tracer 툴을 이용해 분석했을 때는 이 명령은 실제 화면을 갱신하는데는 아무 관련이 없는 것으로 나타나는 것이었습니다.

<p style="text-align: center;">
  (Systrace 툴을 통해서도 확인되는 glTexImage2D 명령)
</p>

![alt text](https://i.imgur.com/Gk68sEw.png "galaxy_s6_glTexImage2D_systrace")

&nbsp;

(Tracer를 통해 확인한 glTexImage2D 명령. 약 4ms(400만 ns) 정도 소요되는 모습. 제일 마지막 glDrawArrays 명령은 화면 상 아무런 변화를 가져오지 않음)

![alt text](https://i.imgur.com/Q3K4axg.png "galaxy_s6_glTexImage2D_tracer")

&nbsp;

&nbsp;

여기까지가 제가 살펴본 내용입니다.

참고 링크 :

[Android Performance Case Study by Romain Guy](http://www.curious-creature.com/docs/android-performance-case-study-1.html)

&nbsp;
