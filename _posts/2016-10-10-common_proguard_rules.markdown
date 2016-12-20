---
layout:     post
title:      "Proguard를 사용한 코드 난독화 - 자주 사용하는 라이브러리의 난독화 규칙, 규칙 분리 관리하기"
subtitle:   "이 규칙을 왜 추가했는지 기억이 나지 않나요? 라이브러리 별로 난독화 규칙을 분리하여 관리해 봅시다."
date:       "2016-10-10 21:30:00+0900"
author:     "커니"
tags:       [android, proguard, minify, rules, library, gson, okhttp3, retrofit2, rxjava, 안드로이드, 프로가드, 난독화, 규칙, 라이브러리]
categories: [lecture, proguard]
---

관련 글

- [Proguard를 사용한 코드 난독화 - 라이브러리 프로젝트에 적용하기]({{ site.baseurl }}/lecture/proguard/2016/07/23/proguard_for_library_project)
- [Proguard를 사용한 코드 난독화 - 모듈별 규칙 적용시 유의사항]({{ site.baseurl }}/lecture/proguard/2016/07/30/library_project_proguard_rules_and_consumer_proguard_rules)
- [Proguard를 사용한 코드 난독화 - 유닛 테스트에서 java.lang.VerifyError가 발생하는 경우]({{ site.baseurl }}/lecture/proguard/2016/08/07/proguard_unit_test_verifyerror)

---

## 프로가드 규칙, 용도와 라이브러리에 따라 분리해서 관리하세요!

프로가드 규칙, 어떻게 관리하시나요?

다음처럼 앱 프로젝트에 하나의 프로가드 규칙을 생성해 두고, 그 안에 한 번에 여러 라이브러리에 필요한 규칙을 한 번에 정의하셨나요?

[proguard-rules.pro]

```
# Don't note duplicate definition (Legacy Apche Http Client)
-dontnote android.net.http.*
-dontnote org.apache.http.**

# Add when compile with JDK 1.7
-keepattributes EnclosingMethod

# Firebase Authentication
-keepattributes *Annotation*

# Firebase Realtime database
-keepattributes Signature

-dontwarn okhttp3.**
-dontwarn okio.**

-dontnote okhttp3.**

# Platform calls Class.forName on types which do not exist on Android to determine platform.
-dontnote retrofit2.Platform
# Platform used when running on RoboVM on iOS. Will not be used at runtime.
-dontnote retrofit2.Platform$IOS$MainThreadExecutor
# Platform used when running on Java 8 VMs. Will not be used at runtime.
-dontwarn retrofit2.Platform$Java8
# Retain generic type information for use by reflection by converters and adapters.
-keepattributes Signature
# Retain declared checked exceptions for use by a Proxy instance.
-keepattributes Exceptions

```

당장 사용하기엔 큰 문제가 없지만, 시간이 지나면 정의한 규칙이 어느 라이브러리, 혹은 어떤 기능 때문에 필요했던 것인지 혼동하기 쉽습니다.

이렇게 하나의 파일에 합쳐져 있는 파일들을 라이브러리, 용도 별로 분리하면 조금 더 용이하게 난독화 규칙을 관리할 수 있습니다.

앞의 예시에서는 안드로이드 플랫폼, 파이어베이스, OkHttp, Retrofit의 난독화 규칙을 하나의 파일에 정의해 두었는데, 이를 우선 다음과 같이 별개의 파일로 분리합니다.

[proguard-common.pro]

```

# Begin: Common Proguard rules

# Don't note duplicate definition (Legacy Apche Http Client)
-dontnote android.net.http.*
-dontnote org.apache.http.**

# Add when compile with JDK 1.7
-keepattributes EnclosingMethod

# End: Common Proguard rules

```

[proguard-firebase.pro]

```

# Begin: Proguard rules for Firebase

# Authentication
-keepattributes *Annotation*

# Realtime database
-keepattributes Signature

# End: Proguard rules for Firebase

```

[proguard-okhttp3.pro]

```

# Begin: Proguard rules for okhttp3

-dontwarn okhttp3.**
-dontwarn okio.**

-dontnote okhttp3.**

# End: Proguard rules for okhttp3

```

[proguard-retrofit2.pro]

```

# Begin: Proguard rules for retrofit2

# Platform calls Class.forName on types which do not exist on Android to determine platform.
-dontnote retrofit2.Platform
# Platform used when running on RoboVM on iOS. Will not be used at runtime.
-dontnote retrofit2.Platform$IOS$MainThreadExecutor
# Platform used when running on Java 8 VMs. Will not be used at runtime.
-dontwarn retrofit2.Platform$Java8
# Retain generic type information for use by reflection by converters and adapters.
-keepattributes Signature
# Retain declared checked exceptions for use by a Proxy instance.
-keepattributes Exceptions

# End: Proguard rules for retrofit2
```

난독화 규칙 파일을 모두 분리했다면, 이를 빌드스크립트에 추가할 차레입니다.

`proguardFile`을 사용하면 각 규칙을 하나씩 추가할 수 있습니다.

다음은 기본 난독화 규칙(proguard-android.txt)과 위에서 정의한 난독화 규칙을 릴리즈 빌드 타입에 적용한 모습을 보여줍니다.

[build.gradle]

```groovy
android {
    buildTypes {
        release {
            minifyEnabled true
            proguardFile getDefaultProguardFile('proguard-android.txt')
            proguardFile 'proguard-common.pro'
            proguardFile 'proguard-firebase.pro'
            proguardFile 'proguard-okhttp3.pro'
            proguardFile 'proguard-retrofit2.pro'
        }
    }
}
```

이렇게 난독화 규칙을 분리해 두면, 차후 라이브러리의 버전을 변경하거나 삭제하는 경우 해당하는 규칙을 쉽게 관리할 수 있습니다.

또한, 다른 프로젝트에 난독화를 적용할 때에도 위에서 분리한 라이브러리별 규칙을 사용할 수 있으므로 대응이 더 편리해집니다.

## 자주 사용하는 난독화 규칙 일람

### 기본 규칙

서드파티 라이브러리를 일체 사용하지 않고, 안드로이드 기본 난독화 규칙 (`proguard-android.txt`)만을 적용했을 때 다음과 같은 메시지를 볼 수 있습니다.

```
Note: duplicate definition of library class [android.net.http.SslCertificate]
Note: duplicate definition of library class [android.net.http.SslError]
Note: duplicate definition of library class [android.net.http.HttpResponseCache]
Note: duplicate definition of library class [android.net.http.SslCertificate$DName]
Note: duplicate definition of library class [org.apache.http.conn.scheme.LayeredSocketFactory]
Note: duplicate definition of library class [org.apache.http.conn.scheme.SocketFactory]
Note: duplicate definition of library class [org.apache.http.conn.scheme.HostNameResolver]
Note: duplicate definition of library class [org.apache.http.conn.ConnectTimeoutException]
Note: duplicate definition of library class [org.apache.http.params.CoreConnectionPNames]
Note: duplicate definition of library class [org.apache.http.params.HttpParams]
Note: duplicate definition of library class [org.apache.http.params.HttpConnectionParams]
```

이는 구 버전의 아파치 HTTP 클라이언트로 인해 발생하는 메시지로, 다음 규칙을 추가하여 메시지를 제거할 수 있습니다.

```
# Don't note duplicate definition (Legacy Apche Http Client)
-dontnote android.net.http.*
-dontnote org.apache.http.**
```

### GSON

매우 많이 사용하는 JSON 라이브러리입니다. 공식 사이트에서 다음과 같은 난독화 규칙을 제공하고 있습니다.

```
# Gson uses generic type information stored in a class file when working with fields. Proguard
# removes such information by default, so configure it to keep all of it.
-keepattributes Signature

# For using GSON @Expose annotation
-keepattributes *Annotation*

# Gson specific classes
-keep class sun.misc.Unsafe { *; }
#-keep class com.google.gson.stream.** { *; }

# Application classes that will be serialized/deserialized over Gson
-keep class com.google.gson.examples.android.model.** { *; }

# Prevent proguard from stripping interface information from TypeAdapterFactory,
# JsonSerializer, JsonDeserializer instances (so they can be used in @JsonAdapter)
-keep class * implements com.google.gson.TypeAdapterFactory
-keep class * implements com.google.gson.JsonSerializer
-keep class * implements com.google.gson.JsonDeserializer
```

하지만, 이 규칙만을 적용하게 되면 난독화 과정 중 다음과 같은 메시지가 표시됩니다.

```
Note: the configuration refers to the unknown class 'sun.misc.Unsafe'
Note: com.google.gson.internal.UnsafeAllocator accesses a declared field 'theUnsafe' dynamically
```

무시해도 무방한 내용이므로, 다음 규칙을 추가하여 메시지를 제거할 수 있습니다.

```
-dontnote sun.misc.Unsafe
-dontnote com.google.gson.internal.UnsafeAllocator
```

### OkHttp3

Retrofit과 함께 주로 사용하는 네트워크 라이브러리입니다. 다음 규칙을 사용하면 됩니다.

```
-dontwarn okhttp3.**
-dontwarn okio.**

-dontnote okhttp3.**
```

### Retrofit2

공식 사이트에서 다음과 같은 난독화 규칙을 제공하며, 아래 규칙만 사용해도 됩니다.

```
# Platform calls Class.forName on types which do not exist on Android to determine platform.
-dontnote retrofit2.Platform
# Platform used when running on RoboVM on iOS. Will not be used at runtime.
-dontnote retrofit2.Platform$IOS$MainThreadExecutor
# Platform used when running on Java 8 VMs. Will not be used at runtime.
-dontwarn retrofit2.Platform$Java8
# Retain generic type information for use by reflection by converters and adapters.
-keepattributes Signature
# Retain declared checked exceptions for use by a Proxy instance.
-keepattributes Exceptions
```

### RxJava

다음 규칙을 추가합니다.

```
-keep class rx.schedulers.Schedulers {
    public static <methods>;
}
-keep class rx.schedulers.ImmediateScheduler {
    public <methods>;
}
-keep class rx.schedulers.TestScheduler {
    public <methods>;
}
-keep class rx.schedulers.Schedulers {
    public static ** test();
}
-keepclassmembers class rx.internal.util.unsafe.*ArrayQueue*Field* {
    long producerIndex;
    long consumerIndex;
}
-keepclassmembers class rx.internal.util.unsafe.BaseLinkedQueueProducerNodeRef {
    long producerNode;
    long consumerNode;
}

-dontwarn rx.internal.**
-dontnote rx.internal.**
```

## 자주 마주하게 되는 메시지에 대한 해결 방법

### the configuration keeps the entry point ..., but not the descriptor class ...

메서드나 필드를 난독화 하지 않도록 하였으나, 인자 혹은 반환형은 난독화를 허용하는 경우 발생합니다.

(예: `void Foo(Bar bar)`에서 `Foo`는 난독화되지 않도록 하였으나 `Bar`는 난독화 될 수 있는 경우)

다음은 메시지의 예를 보여줍니다.

```
Note: the configuration keeps the entry point 'com.arlib.floatingsearchview.FloatingSearchView { void setOnBindSuggestionCallback(com.arlib.floatingsearchview.suggestions.SearchSuggestionsAdapter$OnBindSuggestionCallback); }', but not the descriptor class 'com.arlib.floatingsearchview.suggestions.SearchSuggestionsAdapter$OnBindSuggestionCallback'
```

이렇게 되는 경우 해당 메서드의 시그니처가 변경되므로, 이를 외부에서 호출하는 경우 해당하는 함수를 찾지 못하는 문제가 발생합니다.

이 경우에는 `-keep` 옵션을 사용하여 해당하는 클래스가 난독화 되지 않도록 설정해야 합니다.

만약, 이와 상관이 없다면 다음 규칙을 추가하여 위 메시지를 제거할 수 있습니다.

```
# com.foo.bar 패키지 및 하위 패키지에 대해 메시지를 제거하는 경우
-dontnote com.foo.bar.**
```

참고: [ProGuard Manual](http://proguard.sourceforge.net/manual/troubleshooting.html#descriptorclass)

### Ignoring InnerClasses attribute for an anonymous inner class ... that doesn't come with an associated EnclosingMethod attribute.

JDK 1.7로 컴파일 하는 경우 발생하는 메시지로, 대부분의 프로젝트에서 발생할 것으로 보입니다. (자바 1.8로 컴파일 하는 경우에도 발생하는지 확인해 보진 못했습니다)

```
warning: Ignoring InnerClasses attribute for an anonymous inner class
(h.n) that doesn't come with an
associated EnclosingMethod attribute. This class was probably produced by a
compiler that did not target the modern .class file format. The recommended
solution is to recompile the class from source, using an up-to-date compiler
and without specifying any "-target" type options. The consequence of ignoring
this warning is that reflective operations on this class will incorrectly
indicate that it is *not* an inner class.
```

다음 규칙을 추가하여 해결할 수 있습니다.

```
-keepattributes EnclosingMethod
```

참고: [Stack Overflow](http://stackoverflow.com/questions/26993474/android-dx-warning-ignoring-innerclasses-attribute-for-an-anonymous-inner-class)
