---
id: 428
title: '안드로이드 그래들 레시피 (1) &#8211; 플레이버 디멘션을 이용해 여러 플레이버를 선언하면서도 간편히 테스트하기'
date: 2016-08-28T19:22:43+00:00
author: pungrue26@gmail.com
layout: post
guid: http://www.joooonho.com/?p=428
permalink: '/2016/08/28/%ec%95%88%eb%93%9c%eb%a1%9c%ec%9d%b4%eb%93%9c-%ea%b7%b8%eb%9e%98%eb%93%a4-%eb%a0%88%ec%8b%9c%ed%94%bc-1-%ed%94%8c%eb%a0%88%ec%9d%b4%eb%b2%84-%eb%94%94%eb%a9%98%ec%85%98%ec%9d%84-%ec%9d%b4%ec%9a%a9/'
categories:
  - Uncategorized
---
### 문제 상황 :

&#8211; [안드로이드 테스팅 코드랩](https://codelabs.developers.google.com/codelabs/android-testing/#0)에서 한 것 처럼 [프로덕트 플레이버를 이용해서 테스트](http://android-developers.blogspot.kr/2015/12/leveraging-product-flavors-in-android.html)를 하고 싶다. 그러면서 동시에 프로덕트 플레이버 기능을 다른 용도로도 사용하고 싶다. 이럴 경우 Injection 클래스(진짜 데이터와 가짜 데이터를 적절히 제공하는데 필요한)를 중복해서 만들어야 한다.

### 해결 방법 :

&#8211; 플레이버 디멘션을 이용하면 각 플레이버 디멘션의 목적에 맞는 디렉토리들을 한 번씩만 생성해도 된다.

<!--more-->

* * *

&nbsp;

[안드로이드 개발자 블로그](http://android-developers.blogspot.kr/2015/12/leveraging-product-flavors-in-android.html)와 [구글 코드랩](https://codelabs.developers.google.com/codelabs/android-testing/#0) 등을 통해서 소개된 프로덕트 플레이버를 이용한 테스트는 안드로이드에서 매우 효과적인 테스트 방법입니다. 특히 코드랩에서 구현한 것처럼, MVP 모델 중 View와 Presenter 레벨은 그대로 두고(해당하는 실제 코드가 돌아가고) Model에 해당하는 부분만 가짜로 바꿔치기 함으로써, 예컨대 서버의 응답을 자유롭게 조작하여 테스트해볼 수 있습니다.

그런데 이렇게 구조를 잡는 경우 한 가지 문제를 만나게 됩니다. 만약 프로덕트 플레이버를 이런 테스트 목적이 아니라 다른 용도로도 사용하고자 할 경우 이 둘을 어떻게 조화시키는지가 문제입니다. 예를 들어서, 어떤 앱이 서버로부터 데이터를 받아 사용자에게 보여주는데, 서버의 개발 단계에 따라 3개의 서로 다른 API 엔드포인트가 있다고 합시다.

> 개발 서버: https://**dev**.api.myawesomeapp.com/path?query&#8230;
> 
> 스테이징 서버: https://**stage**.api.myawesomeapp.com/path?query&#8230;
> 
> 실 서버: https://**real**.api.myawesomeapp.com/path?query&#8230;

안드로이드 앱에서 이런 API 주소를 세팅하는 방법은 여러 가지가 있지만, 프로덕트 플레이버를 사용하는 것도 간편한 방법 중 하나입니다. dev, stage, real 의 3개의 플레이버를 선언하고 각각의 경우에 API 주소를 스트링 리소스 형태나 BuildConfig 에 추가함으로써 간단하고 깔끔하게 구현할 수 있습니다.

> 스트링 리소스로 정의하는 방법은 1) [별도의 소스셋을 만들고 여기에 리소스를 추가](http://tools.android.com/tech-docs/new-build-system/user-guide#TOC-Sourcesets-and-Dependencies)하거나, 2) build.gradle 파일에서 플레이버를 선언하면서 [resValue 메서드](http://google.github.io/android-gradle-dsl/current/com.android.build.gradle.internal.dsl.ProductFlavor.html#com.android.build.gradle.internal.dsl.ProductFlavor:resValue(java.lang.String, java.lang.String, java.lang.String))를 이용해 바로 추가할 수도 있습니다. 물론 이 2가지 방법은 내부적으로는 동일하게 동작합니다. 우리가 프로덕트 플레이버를 선언하는 대로 [ProductFlavor 클래스](http://google.github.io/android-gradle-dsl/current/com.android.build.gradle.internal.dsl.ProductFlavor.html#com.android.build.gradle.internal.dsl.ProductFlavor:buildConfigField(java.lang.String, java.lang.String, java.lang.String))의 인스턴스들이 생성되고, 이 각각의 인스턴스가 우리가 추가한 리소스들을 들고 있다가 buildType 소스셋 및 main 소스셋의 리소스들과 우선 순위에 따라 머지됩니다.
> 
> BuildConfig에 선언하고자 할 때는 ProductFlavor 클래스의 [buildConfigField 메서드](http://google.github.io/android-gradle-dsl/current/com.android.build.gradle.internal.dsl.ProductFlavor.html#com.android.build.gradle.internal.dsl.ProductFlavor:buildConfigField(java.lang.String, java.lang.String, java.lang.String))를 이용하면 됩니다.

&nbsp;

그럼 현재까지 5개의 플레이버(mock, prod, dev, stage, real)를 선언하게 되었습니다. **<u>여기서 문제가 되는 것은</u>** 테스트를 하는데 필요한 Injection 클래스를 mock, prod 플레이버 소스셋 뿐만 아니라, dev, stage, real 플레이버의 소스셋에도 추가해야 된다는 점입니다. Injection 클래스를 추가하지 않으면 dev, stage, real 플레이버로 빌드를 하는 경우(devDebug, devRelease, stageDebug, stageRelease 등..) Injection 클래스를 참조할 수 없어서 컴파일 에러가 발생합니다. 그리고 이것은 **리소스 타입과 다르게 java 소스 파일은 오버라이딩이 되지 않기 때문**에 발생합니다. 만약 클래스 오버라이딩이 가능하다면, 위와 같은 상황에서 우리는 main 소스셋과 mock 플레이버 소스셋에만 Injection 클래스를 놓으면 됩니다. 디폴트로는 main 소스셋의 Injection 클래스가 사용(진짜 데이터 모델을 사용하는)되고, mock 플레이버가 사용되는 경우에만 mock 플레이버의 Injection클래스가 사용(가짜 데이터를 사용하는) 되게 될 것입니다. 이런 동작이 리소스에 대해서는 가능하지만 java 소스 파일에 대해서는 가능하지 않습니다. java 컴파일러가 컴파일을 할 때 참조하는 클래스패스에 메인 소스셋과 플레이버 소스셋이 모두 포함되기 때문입니다. 안드로이드 테스팅 코드랩의 소스 코드를 예로 들면 아래 그림과 같은 더러운 구조가 만들어지게 됩니다.

[<img class="aligncenter size-full wp-image-451" src="http://www.joooonho.com/wp-content/uploads/2016/08/스크린샷-2016-08-28-오후-4.37.02-1.png" alt="스크린샷 2016-08-28 오후 4.37.02" width="900" height="992" srcset="http://www.joooonho.com/wp-content/uploads/2016/08/스크린샷-2016-08-28-오후-4.37.02-1.png 900w, http://www.joooonho.com/wp-content/uploads/2016/08/스크린샷-2016-08-28-오후-4.37.02-1-272x300.png 272w, http://www.joooonho.com/wp-content/uploads/2016/08/스크린샷-2016-08-28-오후-4.37.02-1-768x847.png 768w" sizes="(max-width: 900px) 100vw, 900px" />](http://www.joooonho.com/wp-content/uploads/2016/08/스크린샷-2016-08-28-오후-4.37.02-1.png)

위와 같은 프로젝트 구조에서 mock 디렉토리를 제외한, prod, dev, stage, real 디렉토리의 Injection 클래스는 내용이 동일할 것입니다. 내용이 동일한 클래스를 여러 소스셋에 반복해서 만드는 것은 두말할 필요도 없이 해서는 안 되는 구조입니다. 어딘가 수정을 해야 할 때 여러 군데를 반복해서 수정해야 돼서 번거로울 뿐만 아니라 실수를 유발합니다.

&nbsp;

이 문제는 [플레이버 디멘션](http://tools.android.com/tech-docs/new-build-system/user-guide#TOC-Multi-flavor-variants)이라는 기능을 이용해서 간단하게 해결될 수 있습니다. 플레이버 디멘션은 말 그대로 여러 플레이버를 선언하면서, 각각의 플레이버가 어떤 그룹에 속하는지를 추가적으로 지정할 수 있는 기능입니다. 디멘션을 추가하고 나면 각각의 플레이버가 어떤 디멘션에 속하는지도 지정해줘야 합니다. 우리의 프로젝트 구조에서는 다음과 같이 지정할 수 있습니다.

> <pre>flavorDimensions "dataSource", "api"</pre>
> 
> <pre>productFlavors {
    mock { 
        dimension = "dataSource" 
    } 
    prod { 
        dimension = "dataSource" 
    }
    dev {
        dimension = "api"
        resValue "string", "api_base", "https://dev.api.myawesomeapp.com/path?query..."
    }
    stage {
        dimension = "api"
        resValue "string", "api_base", "https://stage.api.myawesomeapp.com/path?query..."
    }
    real {
        dimension = "api"
        resValue "string", "api_base", "https://real.api.myawesomeapp.com/path?query..."
    }
}</pre>

위와 같이 설정하는 경우, 아래와 같이 총 12개의 빌드 베리언트들(빌드 타입 종류 2 \* dataSource 플레이버 3 \* api 플레이버 2)이 생성됩니다.

  1. mockDevDebug
  2. mockDevRelease
  3. mockStageDebug
  4. mockStageRelease
  5. mockRealDebug
  6. mockRealRelease
  7. prodDevDebug
  8. prodDevRelease
  9. prodStageDebug
 10. prodStageRelease
 11. prodRealDebug
 12. prodRealRelease

그리고 프로젝트를 빌드하면 mock\* 베리언트에서는 mock 소스셋의 Injection 클래스가 사용되고, prod\* 베리언트에서는 prod 소스셋의 Injection 클래스가 사용됩니다. 즉, 위와 같은 프로젝트 설정에서 java 클래스패스는 1) main 소스셋 2) dataSource 디멘션 중 현재 베리언트의 소스셋(mock 또는 prod) 3) api 디멘션 중 현재 베리언트의 소스셋(dev, stage, real 중 하나) 이되고, 이 클래스패스들에 들어있는 클래스들로 java 컴파일을 하게 되는 것입니다.

여기서 한 가지 더 언급할 점은, 플레이버 디멘션을 추가할 때 디멘션들이 추가되는 순서도 중요합니다. 위 예시에서 우리는 dataSource 디멘션을 먼저 추가하고, 그 다음 api 디멘션을 추가했는데, 이 순서는 리소스가 머지될 때 어떤 리소스가 우선 순위를 갖는지에 영향을 미칩니다.

마지막으로 위와 같이 플레이버를 추가했을 때 생성되는 빌드 베리언트들 중 의미가 없는 베리언트들을 제거하면 작업이 완성됩니다. mock 플레이버로 빌드를 하는 경우 (mockDevDebug, mockStageRelease 등등..) 어차피 데이터를 서버에 요청하는 것이 아니기 때문에 dev, stage, real 플레이버들은 의미가 없게 됩니다. 따라서 이때는 mockDev{$buildType} 베리언트만 남겨두고 나머지 베리언트들은 제거하면 되겠습니다. 만약 빌드 타입에 따른 구분(debug, release)도 필요 없다면, 적당한 한 개의 베리언트만 남기고 나머지를 모두 제거하시면 됩니다. 베리언트 제거는 android 프로퍼티의 [variantFilter 프로퍼티](http://google.github.io/android-gradle-dsl/current/com.android.build.gradle.AppExtension.html#com.android.build.gradle.AppExtension:variantFilter)를 이용하면 쉽게 할 수 있습니다.

> <pre>variantFilter { variant -&gt;
    def names = variant.flavors*.name
    if (names.contains("mock") && !names.contains("dev")) {
        variant.ignore = true
    }
}</pre>

&nbsp;

이상으로 플레이버 디멘션을 이용해 안드로이드 그래들 플러그인의 프로덕트 플레이버를 좀 더 확장성 있게 사용하는 방법을 알아보았습니다.