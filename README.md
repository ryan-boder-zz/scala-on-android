# Scala on Android

## Why bother?

Android is a Java framework and was intended to run Java apps. Why bother writing Android apps in Scala?

1. We like Scala. Duh.
2. Android is a [huge and growing](https://en.wikipedia.org/wiki/Android_%28operating_system%29#Market_share) ecosystem that runs Java bytecode. Why not leverage it?
3. Simpler Android code. The Android Java API is kind of annoying. The Scala language and the [Scaloid](https://github.com/pocorall/scaloid) and/or [Macroid](https://github.com/47deg/macroid) libraries make it less painful.

## Is it commonly done?

The vast majority of Android apps are written in Java but there appears to be a growing community for Scala on Android. It's hard to find many widely use production Android apps written in Scala.

Android apps written in Scala
- [Tuner & Metronome](https://play.google.com/store/apps/details?id=com.soundcorset.client.android) - by the makers of Scaloid
- [EatIn](https://engineering.linkedin.com/incubator/technology-behind-eatin-android-apps-scala-ios-apps-and-play-framework-web-services) - LinkedIn's internal food rating app
- [Ballero](https://play.google.com/store/apps/details?id=com.cldellow.ballero) - a knit and crochet app
- [Maestroid](https://github.com/ryan-boder/maestroid) - Conduct a symphony with your Android device
- [List of projects using Scaloid](https://github.com/pocorall/scaloid/wiki/Appendix#list-of-projects-using-scaloid)

## Should I do it?

If you want then absolutely. If you're not so sure about it then probably not.

The biggest drawback of using Scala on Android is tooling support. Android officially supports Java development, Scala - not so much. If you're willing to overcome the tooling issues then it's fairly safe.

# Build Tools

Scala projects are usually built with SBT while Android projects built with Gradle.

## SBT Android plugin

If you're going to build with SBT then you'll want the [SBT Android plugin](https://github.com/scala-android/sbt-android). It automates all the Android specific stuff from generating the initial project files to creating and signing the APK artifact.

The SBT Android plugin appears to be most recommended on the web. It even has [IntelliJ](https://www.jetbrains.com/idea) integration through the [IDEA SBT plugin](https://github.com/orfjackal/idea-sbt-plugin).

## Gradle Android Scala plugin

If you prefer a more Android-centric approach then [Gradle Android Scala plugin](https://github.com/saturday06/gradle-android-scala-plugin) might work for you. I haven't tried it but I guess it might play nicer with [Android Studio](https://developer.android.com/studio/index.html) - if you like Android Studio.

## Separate Scala and Android projects

Another approach is to keep your Scala code and your Android code in separate projects and include the Scala project's jar into the Android project.

Pros:
- Removes any tooling difficulties. Use SBT for the Scala project and Gradle for the Android project.
- Cleanly separates your Scala code if it's meant to be portable to platforms other than Android.

Cons:
- More overhead to manage the 2 projects.
- You have to write Java for the shell app that uses the Scala library.

# Testing Tools

Android Scala apps are tested with the usual testing tools such as ScalaTest or JUnit.

## The Android API

The difference between testing Android apps and testing regular Java or Scala apps is that the [Android API](https://developer.android.com/reference/android/package-summary.html) doesn't exist in your development environment's JVM - where your tests run. You can attempt to dependency inject and mock every piece of the API your code uses but it's a lot of work. Worse yet, the Android API makes heavy use of static methods and private constructors so mocking them is difficult.

[Robolectric](http://robolectric.org) is a testing library you can use in your development JVM that provides fake (shadow) implementations of (almost) the entire Android API. It includes a JUnit test runner that will run your tests with the faked Android API loaded and ready to go.

[RoboTest](https://github.com/zbsz/robotest) is an add-on for ScalaTest that provides Robolectric integration for ScalaTest. With RoboTest you can simply mix-in a trait to your ScalaTest class and tests will be run Robolectric's shadow API loaded.

```scala
import android.util.Log
import org.scalatest.{FlatSpec, RobolectricSuite}
class SomeClassSpec extends FlatSpec with RobolectricSuite {
  it should "do what I want" in {
    Log.i("SomeClass", "I'm an Android API static method!")
  }
}
```

## Mocking library

I started with ScalaMock but ended up switching to Mockito because it felt safer mocking Java classes in the Android API with a Java mocking tool.

# Scala Android APIs

## Scaloid

[Scaloid](https://github.com/pocorall/scaloid) is a Scala library that wraps the Android API in a way that make the things you do most often short and sweet. For example:

```scala
// Using the Android API directly in Scala
val button = new Button(context)
button.setText("Greet")
button.setOnClickListener(new OnClickListener() {
  def onClick(v: View) {
    Toast.makeText(context, "Hello!", Toast.LENGTH_SHORT).show()
  }
})
layout.addView(button)
```

```scala
// Using the Scaloid API
SButton("Greet", toast("Hello!"))
```

Scaloid is primarily developed and maintained by [O(n^2)](http://o-n2.com) and used by their [Tuner & Metronome](https://play.google.com/store/apps/details?id=com.soundcorset.client.android) app. It doesn't cover the entire Android API but as the author points out anything missing or that you don't like in Scaloid can be worked around by falling back to using the Android API directly.

## Macroid

[Macroid](https://github.com/47deg/macroid) is Scala DSL for Android GUI's. It's name comes from that fat that it heavily uses Scala macros. Macroid is aggressively optimized as a UI language.

```scala
// Scaloid layout example
new SVerticalLayout {
  STextView("ID")
  val userId = SEditText()
  SButton("Sign in", signin(userId.text))
}.padding(20 dip)
```

```scala
// Macroid layout example
var userId = slot[EditText]

l[VerticalLinearLayout](
  w[TextView] <~ text("ID"),
  w[EditText] <~ wire(userId),
  w[Button] <~ text("Sign in") <~ On.click(signin(userId.get.getText))
) ~> padding(all = 20 dp)
```

Macroid is [well documented](http://macroid.github.io) and even boasts it's [advantages over Scaloid](http://macroid.github.io/differences/Scaloid.html).

Macroid is first and foremost a GUI language while Scaloid aims to be a wrapper to the Android API, including non-GUI parts of the API.

# Scaling Down

Scala applications are usually run in environments where memory space and processing power are plentiful. The standard Scala libraries are big.

Android apps run on mobile devices where resources aren't so plentiful. Furthermore, Android devices don't have the Scala libraries installed centrally so every Scala app would include it's own copy of the Scala libraries.

## ProGuard to the rescue

Android apps are commonly processed with [ProGuard](http://proguard.sourceforge.net) to remove unused code and shrink them down. Using ProGuard is necessary for Scala on Android.

Maestroid APK sizes with ProGuard disabled:
```
leo:maestroid ryan$ ls -l target/android/output/
total 28816
-rw-r--r--  1 ryan  staff  4916876 Jun 12 22:23 maestroid-debug-unaligned.apk
-rw-r--r--  1 ryan  staff  4916881 Jun 12 22:23 maestroid-debug.apk
-rw-r--r--  1 ryan  staff  4914358 Jun 12 22:25 maestroid-release-unsigned.apk
```

ProGuard output during build:
```
Initializing...
Ignoring unused library classes...
  Original number of library classes: 3880
  Final number of library classes:    978
Printing kept classes, fields, and methods...
Shrinking...
Printing usage to [/Users/ryan/Scala/maestroid/target/usage.txt]...
Removing unused program classes and class elements...
  Original number of program classes: 7887
  Final number of program classes:    706
Writing output...
Preparing output jar [/Users/ryan/Scala/maestroid/target/android/intermediates/proguard/classes.proguard.jar]
  Copying resources from program jar [/Users/ryan/.ivy2/cache/org.scaloid/scaloid_2.11/jars/scaloid_2.11-4.2.jar] (filtered)
  Copying resources from program jar [/Users/ryan/.ivy2/cache/org.scala-lang/scala-reflect/jars/scala-reflect-2.11.7.jar] (filtered)
  Copying resources from program jar [/Users/ryan/.ivy2/cache/org.scala-lang/scala-library/jars/scala-library-2.11.8.jar] (filtered)
  Copying resources from program jar [/Users/ryan/.ivy2/cache/org.scalactic/scalactic_2.11/bundles/scalactic_2.11-2.2.6.jar] (filtered)
  Copying resources from program jar [/Users/ryan/Scala/maestroid/target/android/intermediates/classes.jar] (filtered)
[info] Generating dex, incremental=false, multidex=false
[info] dex method count: 5115
[info] Packaged: maestroid-release-unsigned.apk (207.96KB)
```

Maestroid APK sizes with ProGuard enabled:
```
leo:maestroid ryan$ ls -l target/android/output/
total 1264
-rw-r--r--  1 ryan  staff  215086 Jun 12 22:29 maestroid-debug-unaligned.apk
-rw-r--r--  1 ryan  staff  215091 Jun 12 22:29 maestroid-debug.apk
-rw-r--r--  1 ryan  staff  212955 Jun 12 22:31 maestroid-release-unsigned.apk
```

If you're using the SBT Android plugin it will use ProGuard by default. You don't have to do anything to enable it.

## Multidex

In addition to memory hogging the Scala libraries cause another problem with Android applications - they contain more methods than the Android Dalvik Executable (dex) format can support.

The dex format indexes all the methods your app with a 16-bit ID. If you have more than 2^16 = 65,536 methods in your app, including all the libraries it uses, then they don't fit. By merely including the Scala standard libraries your app is too big!

This isn't just a problem for Scala. Many Android apps have more than 64k total methods. Android has provided a solution in a support library called [Multidex](https://developer.android.com/studio/build/multidex.html) that allows an app to have more methods.

Using ProGuard to prune unused code is generally a better solution than enabling Multidex. However, if you are unable to use ProGuard or your app is so large that you hit the 64k limit even with ProGuard then you will need to enable Multidex support.

# Getting Started

Install Android SDK and SBT:
```
$ brew install android-sdk
$ brew install sbt
```

Now that Android SDK is installed use it to create an AVD for the emulator:
```
$ android update sdk
$ android avd
```

Once created, start the emulator:
```
$ android list avd # to see your AVD Names
Available Android Virtual Devices:
    Name: Nexus_5X_API_23
  Device: Nexus 6 (Google)
    Path: /Users/ryan/.android/avd/Nexus_5X_API_23.avd
  Target: Android 6.0 (API level 23)
 Tag/ABI: default/x86
    Skin: 1440x2560
  Sdcard: 100M
$ emulator -avd Nexus_5X_API_23
```

Add to ~/.sbt/0.13/plugins/android.sbt:
```
addSbtPlugin("org.scala-android" % "sbt-android" % "1.6.4")
```

Generate the project files:
```
$ sbt "gen-android android-23 com.mydomain.mypackage MyApp"
```

Edit build.sbt:
```scala
androidBuild

javacOptions in Compile ++= "-source" :: "1.7" :: "-target" :: "1.7" :: Nil

scalaVersion := "2.11.8"
minSdkVersion in Android := "11"

run <<= run in Android

// Copied from the Scaloid hello-scaloid-sbt template project
updateCheck in Android := {} // disable update check
proguardCache in Android ++= Seq("org.scaloid")
proguardOptions in Android ++= Seq("-dontobfuscate", "-dontoptimize", "-keepattributes Signature", "-printseeds target/seeds.txt", "-printusage target/usage.txt"
  , "-dontwarn scala.collection.**" // required from Scala 2.11.4
  , "-dontwarn org.scaloid.**" // this can be omitted if current Android Build target is android-16
  , "-dontwarn scala.xml.**" // Scalactic seems to be triggering XML warnings
)

libraryDependencies ++= Seq(
  "org.scaloid" %% "scaloid" % "4.2",
  "org.scalactic" %% "scalactic" % "2.2.6"
)

libraryDependencies ++= Seq(
  "org.scalatest" %% "scalatest" % "2.2.6" % Test,
  "org.mockito" % "mockito-core" % "1.10.19" % Test,
  "com.geteit" %% "robotest" % "0.12" % Test
)

// Robotest is not thread safe
fork in Test := true
```

Build and run the app on the emulator:
```
$ sbt run
```

# Resources

- [Scala on Android Comprehensive Documentation](http://scala-on-android.taig.io)
- [Macroid Scala on Android Page](http://macroid.github.io/ScalaOnAndroid.html)
- [Scaloid Scala Android Blog](http://blog.scaloid.org)
- [Reflections on starting Android project with Scala](https://blog.scalac.io/2016/05/19/reflections-on-starting-android-project-with-scala.html)
