---
layout:     post
title:      Android Constructor Injection
date:       2019-02-20 09:00:00
summary:    Android Pie has a factory for Android components.
categories: Android
---
Android Pie makes Android developers' work easier. When we get to minSdkVersion 28, we can use an [AppComponentFactory](https://developer.android.com/reference/android/app/AppComponentFactory.html) to use constructor injection on Android components (the classes declared in the manifest: Application, Activity, BroadcastReceiver, Service, ContentProvider). Before, Android would instantiate instances of these classes for us reflectively. This meant we had to do some form of a service lookup, with Android components knowing where their dependencies come from, and do field injection.

The AppComponentFactory for an application is guaranteed to be created by Android once, at the beginning of the application's lifecycle, even before ContentProviders.

Declare the factory as a tag on the application in the manifest.
~~~
<application
  android:appComponentFactory=".ExampleAppComponentFactory"
~~~

Ensure that the AppComponentFactory is public, so Android can create it. Note that any components created by this factory no longer have to be public, since the factory, not Android, now creates them.
~~~
public final class ExampleAppComponentFactory extends AppComponentFactory {
  final AppComponent component = DaggerAppComponent.builder().build();

  @Override public Activity instantiateActivity(ClassLoader cl,
      String className, Intent intent) {
    if (className.equals("com.nightlynexus.appcomponentfactoryexample.ExampleActivity")) {
      return component.exampleActivity();
    }
    throw new AssertionError("Unhandled class: " + className);
  }
}
~~~
I use Dagger 2 to fill my Activity dependency.
For Dagger 2 users, this is as simple as exposing a getter for the Activity instance on the component.
~~~
@AppScope
@Component(modules = AppModule.class)
interface AppComponent {
  ExampleActivity exampleActivity();
}
~~~
My Activity can take its dependencies more easily now.
~~~
final class ExampleActivity extends Activity {
  final OkHttpClient client;

  @Inject ExampleActivity(OkHttpClient client) {
    this.client = client;
  }
}
~~~
The AppComponentFactory takes Android one step closer toward less confusing field injection that is so prevalent in Android. The next step is to conquer [View inflation](https://github.com/square/AssistedInject/).
