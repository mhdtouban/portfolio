+++
date = "2015-09-07T18:02:23+10:00"
draft = false
title = "Dagger2 in the Real World"
blerb = "Let's look at a real world example of how Dagger2 is used in Emergency AUS, and maybe spark some ideas for where Dagger could improve your existing code."
+++

*Dagger* and *Dagger2* may be things you've heard of if you're an Android developer.
Dependency injection has been a hot topic for a while now. Often the examples you
see are solving complex architectural problems, or are too trivial to demonstrate
applicability.

Let's look at a real world example of how *Dagger2* is used in Emergency AUS, and
maybe spark some ideas for where Dagger could improve your existing code.

<img src="../../img/ea-sa-comparison.png" class="post-image-right"/>

[Emergency AUS](//play.google.com/store/apps/details?id=com.gridstone.emergencyaus)
and [Alert SA](https://play.google.com/store/apps/details?id=au.gov.alert.sa) from
[Gridstone](//gridstone.com.au/work/emergency-aus) are two very similar applications.
One provides information on events occurring Australia wide, while the other
provides tailored information for just South Australia. Each app is defined by a
[gradle build flavour](//tools.android.com/tech-docs/new-build-system/user-guide#TOC-Build-Type-Product-Flavor-Build-Variant)
residing in a single project.

Build flavours make it easy to provide simple cosmetic customisations; it's
straightforward to redefine `R.color.primary` and `R.string.app_name` between apps,
but what about logic? In this example, we'll look at the differences in the map
camera behaviour, and how Dagger can make our job of redefining it between apps
easier.

Each application has different logic about how the map should be displayed on first
load. Both attempt to make use of the device's location if it's available, but also
query the events in different ways to determine the appropriate bounds for the
camera to snap to.

One approach might be to simply switch on a query to `BuildConfig.FLAVOR`.

{{< highlight java >}}
void setupMapCamera(Location location) {
  if (BuildConfig.FLAVOR.equals("alertsa") {
    moveCameraForSA(location);
  } else {
    moveCameraForAus(location);
  }
}
{{< /highlight >}}

However, this approach has a few drawbacks. Mainly, we now have code within our
*main* folder that knows about build flavours. We also now have code that's flavour
specific outside of the flavour specific source directories. Ideally, the code that
handles the logic of displaying the camera should reside only in the *emergencyaus*
and *alertsa* folders.

To address this, we could create two utility classes with exactly the same name and
method signatures in our flavour folders, giving us something like...

{{< highlight java >}}
public class MapCameraController {
  public static CameraUpdate getCameraUpdate(Location location) {
    // Flavour specific code here.

  }
}
{{< /highlight >}}

Now our flavour specific logic is housed in the correct directories, but our code
is also particularly error prone. The class and method names must match exactly
between projects (making refactoring a pain). We can't define an interface because
we've chosen to use static methods. If we did want to use an interface we'd have to
adopt a traditional Java singleton pattern...

{{< highlight java >}}
public class FlavouredCameraController implements MapCameraController {
  private static MapCameraController instance = new FlavouredMapCameraController();

  private MapCameraController() {}

  public static MapCameraController getInstance() {
    return instance;
  }

  @Override public CameraUpdate getCameraUpdate(Location location) {
    // Flavour specific code here.

  }
}
{{< /highlight >}}

But now we have to duplicate this factory logic in each flavour, and traditional
singletons just
[plain suck](//stackoverflow.com/questions/137975/what-is-so-bad-about-singletons).
This is where dependency injection comes in and magics our problems away.

I'm going to assume you're familiar with the concepts of *Dagger2* with the following
code. If you're still new to the game,
[this talk from Greg Kick](//www.youtube.com/watch?v=oK_XtfXPkqw) provides a great
introduction.

In each of our flavours, we can define a flavour specific module that provides our
required dependencies. Our Alert SA module might look like...

{{< highlight java >}}
@Module public class FlavorModule {
  @Provides @Singleton MapCameraController provideMapCameraController(SaMapCameraController saController) {
    return saController;
  }
}
{{< /highlight >}}

One of our camera controlling classes can become...

{{< highlight java >}}
@Singleton public class SaMapCameraController implements MapCameraController {
  @Inject SaMapCameraController() {}

  @Override public CameraUpdate getCameraUpdate() {
    // Alert SA specific camera code here.

  }
}
{{< /highlight >}}

...and the code in our *main* folder knows nothing about the disparate behaviour
between flavours. All it needs to do is `@Inject MapCameraController` and Dagger
takes care of providing the correct implementation for the current build flavour.

An argument can be made that we've introduced a lot of boilerplate code just to
get this job done (I haven't even shown the required wiring up of `Components`
that Dagger2 requires). I'd agree if this were the only time you made use of
dependency injection in a project. However, if you're already using it to solve
other problems, you already have most of that boilerplate in place. Now you're
just making use of your existing setup to provide clean resolution of dependencies
on flavour specific logic.

With this setup it's easy for us to define new `MapCameraController`
implementations and change them any time without the client code caring. We also
gain a singleton where:

 - The boilerplate is Dagger's responsibility
 - It can be easily mocked for testing
 - It no longer suffers the drawback of a single global access point

I hope this example sparks some ideas about areas of your code that Dagger could
improve. It's not just a tool for solving
[complex architecture problems](//fernandocejas.com/2015/04/11/tasting-dagger-2-on-android/),
but has applicability in more areas than you might expect.

