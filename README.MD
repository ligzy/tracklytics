# Tracklytics

Tracklytics is an annotation based analytic data delegator. It basically delegates all analytic/tracking data to a tunnel which you can decide what to do with them. 

Common problems for the analytics tools are;
  - Polluted code, more tracking code than the domain code
  - Too many analytic tools to track events and they are spread to code like disease
  - Hard to maintain

Tracklytics moves all these problems to an individual module which has a handler for each analytic tool. By using aspect oriented programming, all tracking codes will be added to your business code during the compile time, which means that you don't need to worry about performance.

## Tracklytics solution
- Moves all analytics/tracking code and add them in compile time.
- Provides a debugging monitor tool to visualise all the triggered events.

<img src='https://github.com/orhanobut/tracklytics/blob/master/art/image4.png'/>

# Usage
Events have a common pattern, and almost all of them contains 2 things.

1. Event name 
2. Attributes (key/value pair)

#### @TrackEvent("EventName")
Trigger the event. This is required for each event. Otherwise nothing will be triggered.
```java
@TrackEvent("title") 
public void trackNoValues() {
  // something
}
```

#### @Attribute("key")
**Scope:** Method, Parameters

Add an attribute by using method parameters. Parameter value will be used as attribute value. Multiple parameter can be used.
```java
@TrackEvent("eventName") 
public void trackMe(@Attribute("eventKey") String eventValue) {
  // something
}
```

Add an attribute by using the return value as attribute value. 
```java
@TrackEvent("eventName") 
@Attribute("eventKey")
public String trackMe() {
  // something
  return "eventValue";
}
```

Use default value when the expected value is null. If the return value or parameter value is null, default value will be used.
```java
@Attribute(value="Login", defaultValue="defaultValue")
```

### FixedAttribute
**Scope:** Method, Class

If the attribute values are constant, use FixedAttribute. 
```java
@TrackEvent("eventName")
@FixedAttribute(key="Login", value="Success")
public void foo(){
}
```
You can also have class-wide attributes. These attributes will be added to each event that is triggered within the class. For example: Following foo() method will also have "Screen Name" attribute.
```java
@FixedAttribute(key="Screen Name", value="Login")
public class LoginActivity{

  @TrackEvent("eventName")
  @FixedAttribute(key="Login", value="Success")
  public void foo(){
  }
}
```

### FixedAttributes
**Scope:** Method, Class

Java 7 or below API's don't allow repeated annotations. Sometimes you may need more fixed attributes. Use this annotation to add multiple attributes
```java
@TrackEvent("eventName")
@FixedAttributes({
  @FixedAttribute(key="Name", value="Something"),
  @FixedAttribute(key="LastName", value="Something")
})
public void foo(){
}
```

### Super Attributes
Some attributes might be used for every event within the app. Both Attribute and FixedAttribute provide this option. Use isSuper flag to keep the attribute in entire application scope. They will be kept in memory and always accessible. They won't be added to current attributes automatically but you will have this option to add. Check TrackingAdapter 
```java
@Attribute(value="key", isSuper=true)
```
or 
```java
@FixedAttribute(key="key", value="value",  isSuper=true)
```

### Note
Any attribute kind can be used at the same time.
```java
@TrackEvent("eventName")
@FixedAttributes({
  @FixedAttribute(key="Name", value="Something"),
  @FixedAttribute(key="LastName", value="Something")
})
@Attribute("key")
public void foo(@Attribute("key1") String value, @Attribute("key2") int code){
  return "value100"
}
```

### TrackableAttribute and Trackable type
```java
class Foo implements Trackable {
  @Override public Map<String, String> getTrackableAttributes() {
    Map<String,String> values = new HashMap<>();
    values.put("key","value");
    return values;
  }
}

@TrackEvent("Event A")
void something(@TrackableAttribute FooTrackable foo){}
```
When Event A is triggered, Foo.getTrackableAttributes() will be added to this event's attributes.

### TransformAttribute
Sometimes you might have some values which can represented in String such Enums, IntDefs. You may need to have the corresponding value in tracking.TransformAttribute helps you in this case. For example: In the following example, position is represented by integer and you want to have a String value which represent exact value such as menu item.

```java
class Foo {
  @TrackEvent("event")
  @TransformAttributeMap(
    keys = {0, 1},
    values = {"value0", "value1"}
  )
  public void foo(@TransformAttribute("key") int position) {
  }
}

// foo(0) : event -> [{"key","value0}]
// foo(1) : event -> [{"key","value1}]
```

### @TrackFilter(100, 200)
Sometimes you may want to track an event for a specific tracker or trackers. TrackFilter will handle it for you. Each TrackingAdapter implementation will ask an id implementation. Tracklytics will use this an identifier to differentiate trackers. ie: Only Tracker id == 100 will track this event.
```java
@TrackFilter(100)
@TrackEvent("title") 
public void trackEventFilter() {
  // something
}
```

### Implement TrackingAdapter for any analytic/tracker tool
Implement TrackingAdapter and initialize them in init method. ie: MixPanel implementation. This part is totally up to you. This is just an example.
```java
public class MixPanelTrackingAdapter implements TrackingAdapter {
  private final MixpanelAPI mixpanelAPI;

  public MixPanelTrackingAdapter(Context context, String apiKey) {
    mixpanelAPI = MixpanelAPI.getInstance(context, apiKey);
  }

  // attributes and superAttributes are self explanatory 
  @Override public void trackEvent(String eventName, Map<String, Object> attributes, Map<String, Object> superAttributes) {
    mixpanelAPI.trackMap(eventName, attributes);
  }

  @Override public void start() {
  }

  @Override public void stop() {
    mixpanelAPI.flush();
  }

  // This will be used as identifier for filter, use different id's for each tracker
  @Override public int id() {
    return 100;
  }

  // Used for debugging tool
  @Override public String toString() {
    return "MixPanel";
  }

}
```

### @Tracklytics

Before using tracklytics, you must initialize it. Create a method which returns Tracker type and initialize all your trackers. All dependencies and other complex logic will be added/handled by tracklytics.
```java
class DefaultApplication extends Application{

  @Tracklytics(TrackerAction.INIT) 
  public Tracker init() {
    return new Tracker.Default().init(
        new MixPanelTrackingAdapter(context, "API_KEY")
    );
  }
}
```

or use static method to init tracklytics
```java
TrackerAspect.init(tracker);
```

For the best usage, use start and stop functions in your activities onStart/onStop or on any other entry/exit points
```java
class MainActivity extends Activity {

  @Tracklytics(TrackerAction.START) 
  @Override void onStart(){
  }
  
  @Tracklytics(TrackerAction.STOP) 
  @Override void onStop(){
  }
  
}
```

## Install
Add the following code block to in your app/build.gradle. 
```groovy
buildscript {
  repositories {
    jcenter()
    maven { url "https://oss.sonatype.org/content/repositories/snapshots/" }
  }
  dependencies {
    classpath 'com.orhanobut.tracklytics:tracklytics-plugin:1.1.11-SNAPSHOT'
  }
}

apply plugin: 'com.android.application'
// must be added after 'com.android.application'
apply plugin: 'com.orhanobut.tracklytics'
```

### Event Debugging Monitor
<img src='https://github.com/orhanobut/tracklytics/blob/master/art/image1.png' height='400'/> <img src='https://github.com/orhanobut/tracklytics/blob/master/art/image2.png' height='400'/>

To be able to use debugging monitor, you need to inject it in the activity class.
```java
 TracklyticsDebugger.inject(this);
```

By clicking "hand" icon, monitor will be displayed and each event will be updated in the list.

Debugging monitor displays
- each event for each tracker
- an option to remove all
- filter option to select tracker, time and keyword (in progress)

### Licence
<pre>
Copyright 2016 Orhan Obut

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

   http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
</pre>
