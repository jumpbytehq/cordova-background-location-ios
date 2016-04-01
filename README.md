# cordova-plugin-mauron85-background-geolocation

## Description

Cross-platform background geolocation for Cordova / PhoneGap with battery-saving "circular region monitoring" and "stop detection".

Plugin is both foreground and background geolocation provider. It is far more battery and data efficient then html5 geolocation or cordova-geolocation plugin. But you can still use it together with other geolocation providers (eg. html5 navigator.geolocation).

## Installing the plugin


```
cordova plugin add https://github.com/jumpbytehq/cordova-background-location-ios

```

## Quick Example

```javascript
document.addEventListener('deviceready', onDeviceReady, false);

function onDeviceReady () {

    /**
    * This callback will be executed every time a geolocation is recorded in the background.
    */
    var callbackFn = function(location) {
        console.log('[js] BackgroundGeoLocation callback:  ' + location.latitude + ',' + location.longitude);

        // Do your HTTP request here to POST location to your server.
        // jQuery.post(url, JSON.stringify(location));

        /*
        IMPORTANT:  You must execute the finish method here to inform the native plugin that you're finished,
        and the background-task may be completed.  You must do this regardless if your HTTP request is successful or not.
        IF YOU DON'T, ios will CRASH YOUR APP for spending too much time in the background.
        */
        backgroundGeoLocation.finish();
    };

    var failureFn = function(error) {
        console.log('BackgroundGeoLocation error');
    };

    // BackgroundGeoLocation is highly configurable. See platform specific configuration options
    backgroundGeoLocation.configure(callbackFn, failureFn, {
        desiredAccuracy: 10,
        stationaryRadius: 20,
        distanceFilter: 30,
        debug: true, // <-- enable this hear sounds for background-geolocation life-cycle.
        stopOnTerminate: false, // <-- enable this to clear background location settings when the app terminates
    });

    // Turn ON the background-geolocation system.  The user will be tracked whenever they suspend the app.
    backgroundGeoLocation.start();

    // If you wish to turn OFF background-tracking, call the #stop method.
    // backgroundGeoLocation.stop();
}
```

NOTE: On some platforms is required to enable Cordova's GeoLocation in the foreground and have the user accept Location services by executing `watchPosition` or `getCurrentPosition`. Not needed on Android.

## Example Application

This plugin hosts a SampleApp in [example/SampleApp](/example/SampleApp) folder. SampleApp can be also used to improve plugin in the future. Read instructions in [README.md](/example/SampleApp/README.md).

## Behaviour

The plugin has features allowing you to control the behaviour of background-tracking, striking a balance between accuracy and battery-usage.  In stationary-mode, the plugin attempts to decrease its power usage and accuracy by setting up a circular stationary-region of configurable `stationaryRadius`. iOS has a nice system [Significant Changes API](https://developer.apple.com/library/ios/documentation/CoreLocation/Reference/CLLocationManager_Class/CLLocationManager/CLLocationManager.html#//apple_ref/occ/instm/CLLocationManager/startMonitoringSignificantLocationChanges), which allows the os to suspend your app until a cell-tower change is detected (typically 2-3 city-block change) Android uses [LocationManager#addProximityAlert](http://developer.android.com/reference/android/location/LocationManager.html). Windows Phone does not have such a API.

When the plugin detects your user has moved beyond his stationary-region, it engages the native platform's geolocation system for aggressive monitoring according to the configured `desiredAccuracy`, `distanceFilter` and `locationTimeout`.  The plugin attempts to intelligently scale `distanceFilter` based upon the current reported speed.  Each time `distanceFilter` is determined to have changed by 5m/s, it recalculates it by squaring the speed rounded-to-nearest-five and adding `distanceFilter` (I arbitrarily came up with that formula.  Better ideas?).

`(round(speed, 5))^2 + distanceFilter`

### distanceFilter
is calculated as the square of speed-rounded-to-nearest-5 and adding configured #distanceFilter.

`(round(speed, 5))^2 + distanceFilter`

For example, at biking speed of 7.7 m/s with a configured distanceFilter of 30m:

`=> round(7.7, 5)^2 + 30`
`=> (10)^2 + 30`
`=> 100 + 30`
`=> 130`

A gps location will be recorded each time the device moves 130m.

At highway speed of 30 m/s with distanceFilter: 30,

`=> round(30, 5)^2 + 30`
`=> (30)^2 + 30`
`=> 900 + 30`
`=> 930`

A gps location will be recorded every 930m

Note the following real example of background-geolocation on highway 101 towards San Francisco as the driver slows down as he runs into slower traffic (geolocations become compressed as distanceFilter decreases)

![distanceFilter at highway speed](/distance-filter-highway.png "distanceFilter at highway speed")

Compare now background-geolocation in the scope of a city.  In this image, the left-hand track is from a cab-ride, while the right-hand track is walking speed.

![distanceFilter at city scale](/distance-filter-city.png "distanceFilter at city scale")

**NOTE:** `distanceFilter` is elastically auto-calculated by the plugin:  When speed increases, distanceFilter increases;  when speed decreases, so does distanceFilter.

## API

### backgroundGeoLocation.configure(success, fail, option)

Parameter | Type | Platform     | Description
--------- | ---- | ------------ | -----------
`success` | `Function` | all | Callback to be executed every time a geolocation is recorded in the background.
`fail` | `Function` | all | Callback to be executed every time a geolocation error occurs.
`option` | `JSON Object` | all |
`option.desiredAccuracy` | `Number` | all | Desired accuracy in meters. Possible values [0, 10, 100, 1000]. The lower the number, the more power devoted to GeoLocation resulting in higher accuracy readings.  1000 results in lowest power drain and least accurate readings. **@see** [Apple docs](https://developer.apple.com/library/ios/documentation/CoreLocation/Reference/CLLocationManager_Class/CLLocationManager/CLLocationManager.html#//apple_ref/occ/instp/CLLocationManager/desiredAccuracy)
`option.stationaryRadius` | `Number` | all | Stationary radius in meters. When stopped, the minimum distance the device must move beyond the stationary location for aggressive background-tracking to engage.
`option.debug` | `Boolean` | all | When enabled, the plugin will emit sounds for life-cycle events of background-geolocation! See debugging sounds table.
`option.distanceFilter` | `Number` | all | The minimum distance (measured in meters) a device must move horizontally before an update event is generated. **@see** [Apple docs](https://developer.apple.com/library/ios/documentation/CoreLocation/Reference/CLLocationManager_Class/CLLocationManager/CLLocationManager.html#//apple_ref/occ/instp/CLLocationManager/distanceFilter).
`option.stopOnTerminate` | `Boolean` | iOS, Android | Enable this in order to force a stop() when the application terminated (e.g. on iOS, double-tap home button, swipe away the app).
`option.startOnBoot` | `Boolean` | Android | Start tracking service on device boot.
`option.startForeground` | `Boolean` | Android | If false location service will not be started in foreground and no notification will be shown.
`option.interval` | `Number` | Android, WP8 | The minimum time interval between location updates in seconds. **@see** [Android docs](http://developer.android.com/reference/android/location/LocationManager.html#requestLocationUpdates(long,%20float,%20android.location.Criteria,%20android.app.PendingIntent)) and the [MS doc](http://msdn.microsoft.com/en-us/library/windows/apps/windows.devices.geolocation.geolocator.reportinterval) for more information.
`option.notificationTitle` | `String` optional | Android | Custom notification title in the drawer.
`option.notificationText` | `String` optional | Android | Custom notification text in the drawer.
`option.notificationIconColor` | `String` optional| Android | The accent color to use for notification. Eg. **#4CAF50**.
`option.notificationIconLarge` | `String` optional | Android | The filename of a custom notification icon. See android quirks.
`option.notificationIconSmall` | `String` optional | Android | The filename of a custom notification icon. See android quirks.
`option.locationService` | `Number` | Android | Set location service provider **@see** [wiki](https://github.com/mauron85/cordova-plugin-background-geolocation/wiki/Android-providers)
`option.activityType` | `String` | iOS | [AutomotiveNavigation, OtherNavigation, Fitness, Other] Presumably, this affects iOS GPS algorithm. **@see** [Apple docs](https://developer.apple.com/library/ios/documentation/CoreLocation/Reference/CLLocationManager_Class/CLLocationManager/CLLocationManager.html#//apple_ref/occ/instp/CLLocationManager/activityType) for more information

Following options are specific to provider as defined by locationService option
### ANDROID_FUSED_LOCATION provider options

Parameter | Type | Platform     | Description
--------- | ---- | ------------ | -----------
`option.interval` | `Number` | Android | Rate in milliseconds at which your app prefers to receive location updates. @see [android docs](https://developers.google.com/android/reference/com/google/android/gms/location/LocationRequest.html#getInterval())
`option.fastestInterval` | `Number` | Android | Fastest rate in milliseconds at which your app can handle location updates. **@see** [android  docs](https://developers.google.com/android/reference/com/google/android/gms/location/LocationRequest.html#getFastestInterval()).
`option.activitiesInterval` | `Number` | Android | Rate in milliseconds at which activity recognition occurs. Larger values will result in fewer activity detections while improving battery life.

Success callback will be called with one argument - location object, which tries to mimic w3c [Coordinates interface](http://dev.w3.org/geo/api/spec-source.html#coordinates_interface).

Callback parameter | Type | Description
------------------ | ---- | -----------
`locationId` | `Number` | ID of location as stored in DB (or null)
`serviceProvider` | `String` | Service provider
`debug` | `Boolean` | true if location recorded as part of debug
`time` | `Number` |UTC time of this fix, in milliseconds since January 1, 1970.
`latitude` | `Number` | latitude, in degrees.
`longitude` | `Number` | longitude, in degrees.
`accuracy` | `Number` | estimated accuracy of this location, in meters.
`speed` | `Number` | speed if it is available, in meters/second over ground.
`altitude` | `Number` | altitude if available, in meters above the WGS 84 reference ellipsoid.
`bearing` | `Number` | bearing, in degrees.


### backgroundGeoLocation.start()

Start background geolocation.

### backgroundGeoLocation.stop()

Stop background geolocation.

### backgroundGeoLocation.isLocationEnabled(success, fail)
NOTE: Android only

One time check for status of location services. In case of error, fail callback will be executed.

Success callback parameter | Type | Description
-------------------------- | ---- | -----------
`enabled` | `Boolean` | true/false (true when location services are enabled)

### backgroundGeoLocation.showLocationSettings()
NOTE: Android only

Show system settings to allow configuration of current location sources.

### backgroundGeoLocation.watchLocationMode(success, fail)
NOTE: Android only

Method can be used to detect user changes in location services settings.
If user enable or disable location services then success callback will be executed.
In case or error (SettingNotFoundException) fail callback will be executed.

Success callback parameter | Type | Description
-------------------------- | ---- | -----------
`enabled` | `Boolean` | true/false (true when location services are enabled)

### backgroundGeoLocation.stopWatchingLocationMode()


#### iOS:

```javascript
backgroundGeoLocation.configure(callbackFn, failureFn, {
    desiredAccuracy: 10,
    stationaryRadius: 20,
    distanceFilter: 30,
    activityType: 'AutomotiveNavigation',
    debug: true, // <-- enable this hear sounds for background-geolocation life-cycle.
    stopOnTerminate: false // <-- enable this to clear background location settings when the app terminates
});
```

## Quirks

### iOS

On iOS the plugin will execute your configured ```callbackFn```. You may manually POST the received ```GeoLocation``` to your server using standard XHR. The plugin uses iOS Significant Changes API, and starts triggering ```callbackFn``` only when a cell-tower switch is detected (i.e. the device exits stationary radius). The function ```changePace(isMoving, success, failure)``` is provided to force the plugin to enter "moving" or "stationary" state.

#### `stationaryRadius`

Since the plugin uses **iOS** significant-changes API, the plugin cannot detect the exact moment the device moves out of the stationary-radius.  In normal conditions, it can take as much as 3 city-blocks to 1/2 km before stationary-region exit is detected.


## Debugging sounds
|    | *ios* | *android* | *WP8* |
| ------------- | ------------- | ------------- | ------------- |
| Exit stationary region  | Calendar event notification sound  | dialtone beep-beep-beep  | triple short high tone |
| GeoLocation recorded  | SMS sent sound  | tt short beep | single long high tone |
| Aggressive geolocation engaged | SIRI listening sound |  | |
| Passive geolocation engaged | SIRI stop listening sound |  |  |
| Acquiring stationary location sound | "tick,tick,tick" sound |  | double long low tone |
| Stationary location acquired sound | "bloom" sound | long tt beep | double short high tone |  

**NOTE:** For iOS  in addition, you must manually enable the *Audio and Airplay* background mode in *Background Capabilities* to hear these debugging sounds.

## Changelog

See [CHANGES.md](/CHANGES.md)
