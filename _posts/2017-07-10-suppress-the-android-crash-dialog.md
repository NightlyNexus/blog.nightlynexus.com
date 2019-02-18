---
layout:     post
title:      Suppress the Android Crash Dialog
date:       2017-07-10 08:00:00
summary:    Tricks when suppressing the system crash dialog.
categories: Android
---
[![](/images/crash-dialog.jpg)](https://twitter.com/volumedownpower/status/588170076974620673)

I was listening to [Py's talk about crashes](http://fragmentedpodcast.com/episodes/88/) recently, and he mentioned the hack of suppressing the Android crash dialog.
This system dialog serves to set the user expectation that the app is not functioning properly.
But, sometimes, the crash dialog can be more annoying than helpful to the user.
While every developer should, of course, strive to fix the source of the crash, here is how one might go about suppressing the system crash dialog.
~~~
Thread.setDefaultUncaughtExceptionHandler((thread, e) -> {
  new Handler(Looper.getMainLooper()).postAtFrontOfQueue(() -> Runtime.getRuntime().exit(0));
});
~~~
Of note here is that we post the exit call to the next frame. If we don't, and the crash happens before the first frame is rendered (e.g. in Activity.onCreate), Android will continuously try to restart the current foreground Activity (and the app will probably keep crashing at the same place). On older Android versions, the app will become unresponsive if it continuously crashes and Android tries to restart the Activity. On new Android versions, the system will display a dialog to the user that the app keeps crashing and offer to suppress further notices. So, posting to the next frame is a good trick.

Also, if you use any crash reporting tools that make use of setting the default UncaughtExceptionHandler, make sure to install the UncaughtExceptionHandler above first, so that you don't miss the crash in your other reporting tools.

The crash dialog is less helpful to the user when the app is in the background, so let's suppress it only for background crashes.
~~~
ForegroundChecker foregroundChecker = ForegroundCounter.createAndInstallCallbacks(application);
final Thread.UncaughtExceptionHandler defaultHandler = Thread.getDefaultUncaughtExceptionHandler();
Thread.setDefaultUncaughtExceptionHandler((thread, e) -> {
  if (foregroundChecker.inForeground()) {
    defaultHandler.uncaughtException(thread, e);
  } else {
    new Handler(Looper.getMainLooper()).postAtFrontOfQueue(() -> Runtime.getRuntime().exit(0));
  }
});
~~~
where a ForegroundChecker can be just as simple as checking for any "started" (visible window) Activities:
~~~
final class ForegroundChecker implements ActivityLifecycleCallbacks {
  static ForegroundCounter createAndInstallCallbacks(Application application) {
    ForegroundChecker checker = new ForegroundChecker();
    application.registerActivityLifecycleCallbacks(checker);
    return checker;
  }

  int startCount;

  boolean inForeground() {
    return startCount != 0;
  }

  @Override public void onActivityStarted(Activity activity) {
    startCount++;
  }

  @Override public void onActivityStopped(Activity activity) {
    startCount--;
  }
}
~~~

Finally, I suggest putting this extra UncaughtExceptionHandler only in production builds, so internal builds will still see the crash dialog all the time.

This is a neat trick that some popular apps employ when dealing with large legacy code bases. Crash dialogs annoy users but also set expectations, so use this sparingly. Happy crashing!
