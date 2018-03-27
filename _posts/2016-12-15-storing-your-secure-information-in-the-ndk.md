---
title: Storing your secure information in the NDK
author: Marcos Placona
image: /images/android-ndk.png
---

Reverse engineering and tampering can be easily accomplished in Android. There are measures you can take to [stop hackers from tampering with your Android applications]({% post_url 2016-12-07-tampering-detection-in-android %}), but ultimately a determined hacker will always have the last say.

Let's look at the following code which has been packaged into a signed `*.apk`.

```java
public class MainActivity extends AppCompatActivity {
    private static final String TAG = "MainActivity";
    private static final String AUTH_TOKEN = "V293ISBob3cgY3VyaW91cyBlaD8=";

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // Some super API call using that key
        Log.i(TAG, "key: " + AUTH_TOKEN);
    }
}
```  

It's is a common for developers to store their authentication keys to external services inside their application. This poses a huge security risk since malicious users can decompile the app and get access to said keys in about 10 seconds. 

![Decompiling an Android app](/images/smalify.gif)

Java code can be easily [decompiled](https://en.wikipedia.org/wiki/Decompiler). C++ on the other hand can't be decompiled but can be [disassembled](https://en.wikipedia.org/wiki/Disassembler), which is slightly less trivial.

We will use this advantage and the fact that Android comes with an `NDK` ([Native Developer Toolkit](https://developer.android.com/ndk/index.html)) that helps us write code in C++ and use it from within our application.

## What we'll need
- Android Studio with an installed NDK and Build Tools. Find out how to install those things [here](https://developer.android.com/studio/projects/add-native-code.html#download-ndk).
- An application you've already built. I will be using [this Hello World application](https://github.com/mplacona/HelloWorld) I put together to keep it simple.

## Get coding
To make things simpler on Android Studio, switch the editor view from `Android` to `Project`.

![Switch Android Studio View](/images/switch-to-project-view.png)

Create a new directory called `cpp` under `app/src/main`. Inside that new directory create a `C/C++ Source File` called `native-lib.cpp`.

We now need to tell our project that we want to use and build that library with our code.

We do that by [creating a CMake build script](https://developer.android.com/studio/projects/add-native-code.html#create-cmake-script) under `app/src` and calling it `CMakeLists.txt`.

In that file we will specify what we're building and where our source files are.

```cpp
# Sets the minimum version of CMake required to build your native library.
# This ensures that a certain set of CMake features is available to
# your build.

cmake_minimum_required(VERSION 3.4.1)

# Specifies a library name, specifies whether the library is STATIC or
# SHARED, and provides relative paths to the source code. You can
# define multiple libraries by adding multiple add.library() commands,
# and CMake builds them for you. When you build your app, Gradle
# automatically packages shared libraries with your APK.

add_library( # Specifies the name of the library.
             native-lib

             # Sets the library as a shared library.
             SHARED

             # Provides a relative path to your source file(s).
             src/main/cpp/native-lib.cpp )
```

Right click on `app/src/main/cpp/native-lib.cpp` and choose `Link C++ Project with Gradle`. Leave the `Build System` as `CMAKE` and choose the location for `CMakeLists.txt` as the `Project Path`.

Gradle will sync automatically and once that's done, open up your `MainActivity` or wherever you want to load the secure information and add the following to the top of the class:

```java
static {
    System.loadLibrary("native-lib");
}

private native String invokeNativeFunction();
```

Android studio will now complain that the [JNI](https://en.wikipedia.org/wiki/Java_Native_Interface) Function you're trying to call doesn't exist. Let it go ahead and create that function for you.

Change that function to:

```cpp
#include <jni.h>

extern "C" {
    JNIEXPORT jstring JNICALL
    Java_info_androidsecurity_helloworld_MainActivity_invokeNativeFunction(JNIEnv *env, jobject instance) {
        return env->NewStringUTF("V293ISBob3cgY3VyaW91cyBlaD8=");
    }
}
```

We've added the `extern` declaration so the JVM can see our function and are now returning the key from this JNI function.

Now we just have to call that function within our code just like we did before:

```java
@Override
protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    setContentView(R.layout.activity_main);

    Context mContext = getApplicationContext();

    // Some super API call using that key
    Log.i(TAG, "key: " + invokeNativeFunction());
}
```

Now even if you decompile this `*.apk` you will see that the [smali](https://github.com/JesusFreke/smali) generated does not have the key, and the only way to try and work it out would be by disassembling the library and opening it in a hex editor, but you would really need to know exactly what you are looking for.

 ![Disassembled library](/images/disassemble-lib.png)

### Verdict & Effectiveness
While it is still possible to get information stored inside the app, storing it in the NDK makes it way harder for inexperienced hackers to easily get hold of your information.

This methodology combined with encryption would make it significantly hard for a malicious user to work out where to get information from and how to use it. A commonly used technique is to also convert the key into hexadecimal as it would make it less obvious when using a hex editor.

<u>Effectiveness:</u> <i class="fa fa-battery-full" aria-hidden="true"></i>

## <i class="fa fa-file-code" aria-hidden="true"></i>
You can see all the code samples shown here in [this repository](https://github.com/mplacona/HelloWorld/tree/SecureJNI).