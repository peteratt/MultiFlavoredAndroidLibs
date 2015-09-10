So I was dabbling today with an app that keeps getting bigger and bigger. So big that my Android library module's DEX had grown into the dreaded 65K method limit [1], and I wanted to optimize it because builds were taking ages [2]. So I went on and created two flavors of my app:

* `dev:` build-time optimized, using modern versions of Android.

* `prod:` compatibility optimized, with compatibility modes active.

Problem was, my `app` module completely stopped picking up the lib when I tried to sync with Gradle. After a lot of investigation, I reached the conclusion that I needed a compile step for both `dev` and `prod` flavors.

####Before

`mylib/build.gradle:`

```
android {
    productFlavors {
        dev {
            // Your dev tweaks here
        }
        prod {
            // Your prod tweaks here
        }
    }
}
```

`app/build.gradle:`
```
dependencies {
    compile ':mylib'
}
```

####After

`mylib/build.gradle` stays the same, but...

`app/build.gradle:`
```
dependencies {
    devCompile project(path: ':mylib', configuration: 'devDebug')
    prodCompile project(path: ':mylib', configuration: 'prodDebug')
}
```

This solution, even though it works, is not optimal. We are losing one very important dimension in our build variants. As you can see, for both `debug` and `release` build types, `app` will take the `debug` build type of `mylib` all the time. This is obviously not very good, since we'd lose all the possible optimizations we can include while building our library for release (e.g. Proguard).

###Even better: adding custom configs

The solution to this issue is adding a custom config inside `app/build.gradle`, the following way:

```
configurations {
    devDebugCompile
    prodDebugCompile
    devReleaseCompile
    prodReleaseCompile
}

dependencies {
    devDebugCompile project(path: ':mylib', configuration: 'devDebug')
    prodDebugCompile project(path: ':mylib', configuration: 'prodDebug')

    devReleaseCompile project(path: ':mylib', configuration: 'devRelease')
    prodReleaseCompile project(path: ':mylib', configuration: 'prodRelease')
}
```

By adding our custom configurations, we can support all build variants. This is easily extensible to many other cases in which you might find a need to support multiple app/library flavors (e.g. free app vs. paid app). You can easily select your build variants in Android Studio and it'll all work like a charm.

{<1>}![Build Variants](/content/images/2015/Sep/Screen-Shot-2015-09-10-at-6-28-59-PM.png)

Just with a little bit of Gradle code, we achieved quite a few things:

* **Multidex support.** 65K-method limit is not a lot, and if you have a few big library dependencies.
* **Optimized dev builds.** I measures Gradle times for building `mylib` equivalents with and without `productFlavors` and it makes the difference between a couple seconds and three minutes, at least on my laptop. We can work at an optimized environment without hassle.
* **Flexible builds**. We can mix-and-match flavors inside libraries, apps. Lots of variants for achieving the goal of a tight, single-codebase project [3].

**Sample app available on Github: [https://github.com/peteratt/MultiFlavoredAndroidLibs](https://github.com/peteratt/MultiFlavoredAndroidLibs)**

[1] http://developer.android.com/tools/building/multidex.html

[2] http://developer.android.com/tools/building/multidex.html#dev-build

[3] http://tools.android.com/tech-docs/new-build-system/build-system-concepts
