
The new Steam Link app did something interesting: it started as a 2D app, but then launched into VR.

  

Meta Quest always provided a way to run “flat” 2D Android phone/tablet applications inside the Quest Home environment – no VR-specific application code required.

  

However, Quest doesn’t usually have hybrid apps: e.g. an app that starts as a 2D UI inside the Quest Home environment, and subsequently transitions into its own app-rendered environment at a later stage.

  

It would almost certainly be useful: if applications could leverage the Quest 2D container for certain activities – i.e. showing interactive forms or deep-linking to web content – they could stand-up well-integrated, professional-looking 3D interfaces right away, and even re-use components across VR- and non-VR versions of their app.

  

If you as an app developer tried to do what Steam Link did, you’d probably struggle. Some developers were asking how to make an app do this. As is sometimes the case, I don’t know if I know the right way, but I managed to figure out *a* way after a lot of experimentation.

  

  

# Objective: Launch a VR Activity from a 2D Activity (without changing applications)

If you try this without any additional instruction, the VR activity will show up as a black rectangle inside Quest Home. We don’t want black inside a rectangle: we want a 3D scene covering both our eyes!

There’s a few things I needed to do to get it working:

## VR Activity Launch
```java
static void launchVrActivity() {
	// 0. Create the VrActivity launch intent
	Intent intent = new Intent(context, VrActivity.class);  
	
	// 1. Locate the main display ID and add that to the intent
	final int mainDisplayId = getMainDisplay(context);  
	ActivityOptions options = ActivityOptions.makeBasic().setLaunchDisplayId(mainDisplayId);  
	
	// 2. Set the following flags: start VrActivity in a new task and replace any existing tasks in the app stack
	intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK | Intent.FLAG_ACTIVITY_CLEAR_TASK | 		Intent.FLAG_ACTIVITY_CLEAR_TOP | Intent.FLAG_ACTIVITY_SINGLE_TOP);    
	
	// 3. Launch the activity.
	// Don't use the container's ContextWrapper, which is adding arguments -- use the base contenxt
	if (context instanceof ContextWrapper) {  
		ContextWrapper contextWrapper = (ContextWrapper) context;  
		Context baseContext = contextWrapper.getBaseContext();  
		baseContext.startActivity(intent, options.toBundle());  
	} else {  
		context.startActivity(intent, options.toBundle());  
	}  
	
	// 4. Finish the previous activity: this avoids an audio bug specific launching a new activity outside the 2D window. Can be omitted if you value preserving 2DActivity over not-having audio bugs.
	((Activity)(context)).finish();
}

static int getMainDisplay(Context context) {  
	final DisplayManager displayManager =  
	(DisplayManager) context.getSystemService(Context.DISPLAY_SERVICE);  
	Display[] displays = displayManager.getDisplays();  
	for (int i = 0; i < displays.length; i++) {  
		if (displays[i].getDisplayId() == Display.DEFAULT_DISPLAY) {  
			return displays[i].getDisplayId();  
		}  
	}  
	return -1;  
}
```

**Explanation**
Normally in Android, all you would have to do to launch ActivityB from ActivityA is:
`startActivity(new  Intent(this, ActivityB.class));`

However, in the hybrid app case, this causes the VrActivity to be launched as a black rectangle inside the 2D app container.
In logcat, it was clear that this was because `VrActivity` was launching on some random display, likely part of the container: it needs to launch on the primary display, Display 0.

1. On Android, activities launch on the same display as their parent. Setting  `LaunchDisplayId` in activityOptions overrides this behavior.
2. In addition to setting the display ID, it was important to specify that the new activity was not part of the same app stack and process as the original activity
3. When I checked, I saw that activities launched in the 2D container view have a ContextWrapper around their context -- this is something the system is adding. Removing it was necessary to break out of the container
4. This would be a "bug" if this behavior was actually supported: the audio system seems to work such that, if a VR activity backgrounds a 2D container activity that shares the same package name, the VR Activity's audio streams (some of them) are muted until the display is turned off/on. However, finishing the first activity prevents this.


## AndroidManifest.xml:

Normally, the `AndroidManifest` declaration for a VR activity looks like this:
```xml
	<activity
            android:name=".MainActivity"
            android:configChanges="density|orientation|screenSize|keyboard|keyboardHidden|uiMode"
            android:excludeFromRecents="true"
            android:exported="true"
            android:label="@string/app_name"
            android:launchMode="singleTask"
            android:resizeableActivity="false"
            android:screenOrientation="landscape"
            android:theme="@android:style/Theme.NoTitleBar.Fullscreen">

            <!-- The "VR" category is needed to make the app launch in
                 "VR mode" (instead of 2D).

                 Fun fact: the XR samples are getting
                 away with not setting this because
                 they are setting `com.samsung.android.vr.application.mode` in the
                 metadata. However, the samsung-mode way isn't in-line with
                 current app store submission guidelines.  -->
            <intent-filter>
	            <category android:name="com.oculus.intent.category.VR" />
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

```
My 2D and VR activity manifest entries now look like this:
```xml
<activity  
	android:name=".2DActivity"  
	android:exported="true"  
	android:resizeableActivity="false">  
	<intent-filter>  
		<action android:name="android.intent.action.MAIN" />  
		<category android:name="android.intent.category.LAUNCHER"/> 
	</intent-filter>  
</activity>

<activity  
	android:name=".VrActivity"  
	android:configChanges="density|orientation|screenSize|keyboard|keyboardHidden|uiMode"  
	android:exported="true"  
	android:launchMode="singleTask"  
	android:resizeableActivity="false"  
	android:screenOrientation="landscape"  
	android:process=":vr_process"  
	 android:theme="@android:style/Theme.NoTitleBar.Fullscreen">  
	<intent-filter>  
		<category android:name="com.oculus.intent.category.VR" />  
		<category android:name="android.intent.category.LAUNCHER" />  
	</intent-filter>  
</activity>
```
**Key Changes:**
1. **VrActivity**: Remove `<action  android:name="android.intent.action.MAIN"  />` to `intent-filter`.
      - However, keep `<category  android:name="android.intent.category.LAUNCHER"  />`,  because the XR runtime appears to depend on it (app submission guidelines require it).
2. **2DActivity** Add `<action android:name="android.intent.action.MAIN" />`  and `<category android:name="android.intent.category.LAUNCHER"/> ` to  to `intent-filter`for the 2D Activity
2. **android:process=** add `android:process=":vr_process"`
   - Note: this is absolutely required, but will make the new activity launch as a separate process, which will have different behavior from a normal activity launch (e.g. separate memory space). Not a concern if the previous activity finishes on launch.

