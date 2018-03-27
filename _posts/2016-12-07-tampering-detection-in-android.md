---
title: Tampering Detection in Android
author: Marcos Placona
---

Tampering detection is a preventive measure used in mobile applications to help ensuring that a third party hasn't recompiled and published your application under their account or store without your consent. 

In this article we will look at two different approaches do detect and mitigate the tampering of Android applications.

* [Check if application has been renamed.](#renamed)
* [Check if application has been published without your consent.](#published)

## <a name="renamed"></a>Check if application has been renamed
When you create a new Android Application you're asked to give it a package name, which in Java is really your reverse domain along with the application name. If we were creating a new `Hello World` application here, we would probably end up having a package name called `info.androidsecurity.helloworld`.

![Package Names in Android](/images/tampering-detection-in-android-01.png)

In an existing application we can use `getApplicationContext().getPackageName()` to return the package name for that application.

With that, we can now check that if our application was renamed to anything else, we'd have the chance to do something about it, or at the very least inform the user that they may not be running the original version of it.

To do that, we can store the name of our package within our application, and then check that every time the application is run.

```java
private static final String PACKAGE_NAME = "info.androidsecurity.helloworld";
//...
if(mContext.getPackageName().compareTo(PACKAGE_NAME) != 0){
    Log.d(TAG, "this is a hack");
    /*
        Things you could do here:
        - show an alert to the user and refuse to proceed
        - make an HTTP request to your own server and alert you
    */            
}
```

### Verdict & Effectiveness
We have very quickly added simple tampering detection to our app. Depending on the approach we take when tampering is detected, we can as much as get alerted about tampering in real-time.

Adding this kind of tampering detection adds very small footprint to your code, and will help stopping hackers who aren't absolutely determined they should break into your code.

That is because the approach above uses string comparison and things can get decompiled and modified fairly easily. 

Usually it's a good idea to store information like `PACKAGE_NAME` in a different class, encrypted or on a remote server. In this case however if a hacker went through the trouble of decompiling our app, they could very easily just delete the bit of logic that checks for the package name.

<u>Effectiveness:</u> <i class="fa fa-battery-quarter">

## <a name="published"></a>Check if application has been published without your consent.
It's very common to have Android applications republished on alternate markets or their `APK`s made available for download.

For a hacker, it's much easier to tamper with an APK than to download it directly from a store via their devices. Here's how to check if a user is trying to run your APK directly without downloading it from a store:

```java
String installer = mContext.getPackageManager().getInstallerPackageName(PACKAGE_NAME);
if (installer == null){
    Log.d(TAG, "this is an APK");
    /*
        Things you could do here:
        - show an alert to the user and refuse to proceed
        - make an HTTP request to your own server and alert you
    */
}
```

If the installer is `null`, the user didn't download this app via conventional methods and most likely got the APK straight from an alternate resource. We would have expected it to be something like `com.android.vending` or `com.amazon.venezia` for Google and Amazon respectively. 

**But what happens if the value of `installer` is not null?**

*Good question!*

Alternate stores will return their own package name when a user downloads your app from them, so just checking that the `installer` is `null` won't always work. The next thing we can do with this code is also check that the installer is one we published our app to. 

```java
if (installer.compareTo(GOOGLE_PLAY) != 0 && installer.compareTo(AMAZON_STORE) != 0){
    Log.d(TAG, "not installed from either stores");
    /*
        Things you could do here:
        - show an alert to the user and refuse to proceed
        - make an HTTP request to your own server and alert you
    */
}
```

This will give you the option to act upon the fact that the users has not downloaded your app from one of the stores you chose to trust.

### Verdict & Effectiveness
With this second level of tampering detection in our app, we make it even harder for hackers to recompile it.

Just like the other approach, we are using string comparison and **again** things can get decompiled and modified fairly easily. 

<u>Effectiveness:</u> <i class="fa fa-battery-quarter">

## Bonus
Since we've created a way to check whether the application was downloaded as an APK or from an alternate store, it's only right that we get those two approaches together into one that will check for everything. We can create a method like this in our utility class.

```java
public boolean isHacked(Context context, String myPackageName, String google, String amazon)
{
  //Renamed?
  if (context.getPackageName().compareTo(myPackageName) != 0) {
      return true; // BOOM!
  }

  //Relocated?
  String installer = context.getPackageManager().getInstallerPackageName(myPackageName);

  if (installer == null){
      return true; // BOOM!
  }

  if (installer.compareTo(google) != 0 && installer.compareTo(amazon) != 0){
    return true; // BOOM!
  }
  return false; 
}
```

And now you just need to call the method `isHacked` from your `onCreate` method in your initial `Activity` to check that your application was renamed or relocated.

<u>Effectiveness:</u> <i class="fa fa-battery-half">

## <i class="fa fa-file-code" aria-hidden="true"></i>
You can see all the code samples shown here in [this repository](https://github.com/mplacona/HelloWorld).