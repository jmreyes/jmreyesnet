---
title: "Modern Android testing: collected resources"
date: 2015-11-11T10:54:54+02:00
---

Some notes to self and whoever finds them useful about the state of Android testing as of late 2015. I keep running into old articles that, while having useful information, prove confusing most of the time, so this is an attempt at collecting the most useful pieces of information that I've found recently.

During this this last year Google has added built-in testing support for Android Studio along with a revamped Android Testing Support Library. One consequence to this has been the fact that we now have something closer to what is usually searched for as *best practices* for Android testing. The three main test categories follow, along with the directory where Android Studio is expecting them by default:

* **JVM unit tests with JUnit4**
  * Those that will be run directly on the Java Virtual Machine. The objects under test will need to be POJOs. Android framework classes, if needed, must be mocked.
  * Directory *test* under *src*.
* **Instrumented unit tests with AndroidJUnit4**
  * Those that will be run over the real Android framework, but with no UI. Can reduce mock code. Must be run on emulator or real device.
  * Directory *androidTest* under *src*.
* **Instrumented UI tests with Espresso[^n]**
  * Those that will be run over the real Android framework, launching a certain Activity and automate user actions over the UI. Must be run on emulator or real device.
  * Directory *androidTest* under *src*.

Keeping that in mind, we can start deepening our knowledge about them. And, now the bulk of the post, I found the following resources pretty useful to do so (order is relevant from my point of view):

### Kickstarting Android testing

1. [Android Testing Codelab](http://www.code-labs.io/codelabs/android-testing/index.html). *Official*. A really nice example that walks you through the implementation of a basic testable app where you will write unit tests with JUnit4 (using Mockito where necessary) and instrumented tests with Espresso. It also works as an introduction to the MVP pattern and its advantages for testability.
2. [Android Testing Support Library Documentation and Samples](https://google.github.io/android-testing-support-library/). *Official*. More detailed information about Espresso, including a useful [cheat sheet](https://google.github.io/android-testing-support-library/downloads/espresso-cheat-sheet-2.1.0.pdf).

### Tech talks

1. [Supercharging Your Android Testing](https://realm.io/news/ellen-shapiro-android-testing/) by [Ellen Saphiro](https://www.twitter.com/designatednerd). Focused on instrumentation tests with AndroidJUnit4 and Espresso, includes live demos.
2. [Advanced Android Espresso](https://www.youtube.com/watch?v=GlPn60-_txk) by [Chiu-Ki Chan](https://twitter.com/chiuki). Advanced Espresso topics, like its use with RecyclerViews and interoperability with Dagger.

### Sample code repos

1. [Samples](https://github.com/chiuki/espresso-samples) and [Friendspell app](https://github.com/chiuki/friendspell) from Chiu-Ki Chan. Great complement to her talk. Some pretty advanced stuff (especially the second one). Includes useful examples for IdlingResource, RecyclerView, and testing screen rotation. The app includes testing of external frameworks like Google Plus login and Nearby API.

[^n]: To be fair, Espresso leverages AndroidJUnit4.
