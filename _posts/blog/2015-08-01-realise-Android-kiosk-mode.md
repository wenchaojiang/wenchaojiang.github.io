---
layout: post
title: Making an Android Kiosk App.
categories: blog
tags: [Android,Kiosk]
image:
  feature:
comments: true
share: true
date: 2015-08-01T15:39:55-04:00
---
### Introduction
Motivation of this article is to write down how to realise a Kiosk mode App, through which a user locked in and can not mess around with other system functionalities. My approach does not require root.

My kiosk mode has several requirement:

  1. lock the user in the app.
  2. boot into the app when system start up.
  3. the app should be full screen.
  4. keep system awake all time.
  5. disable volume buttons.

My approaches:

  1. use app pinning feature newly introduced in SDK level 22 to lock the user in the app.
  2. set the app as a home intent, so system can boot into the app.
  3. make the app full screen with immersive mode of activity.
  4. keep system awake through sdk
  5. disable the volume buttons through sdk

Next I will introduce how to do it step by step, all the template code is available in the repo:
[AndroidKioskModeTemplate](https://github.com/wenchaojiang/AndroidKioskModeTemplate)

### Configurations
You need target Sdk level 21 for screen pinning feature.

### Screen pinning
Andriod 5.0 introduces the screen pinning feature, as its documentation said:
 Once your app activates screen pinning, users cannot see notifications, access other apps, or return to the home screen, until your app exits the mode.

The screen pinning mode can be simply calling  ```startLockTask()``` in your activity and exit by calling ```stopLockTask()```.

However, when ```startLockTask()``` called, system will ask user the permission for enterring pinning mode and user can still exit the pinning mode by touch "back and task" buttons all together. The is not "true" Kiosk that lock user in. To enable a true kiosk mode, your app need to call the ```startLockTask()``` as a device owner.

To set your app as device owner, you need to implement a ```DeviceAdminReceiver``` class ```MyAdmin```, and use the dpm tool under adb shell

#### Implementing a DeviceAdminReceiver
First, create a class inheriting the ```DeviceAdminReceiver```, no implemenation is required.
{% highlight java  %}
package wenchao.kiosk;

import android.app.admin.DeviceAdminReceiver;
public class MyAdmin extends DeviceAdminReceiver{

}
{% endhighlight %}

Second, declare the receiver in ```Manifest.xml```, in your ```<applicatio>```n element

{% highlight xml  %}
<receiver android:name="wenchao.kiosk.MyAdmin"
            android:label="@string/sample_device_admin"
            android:description="@string/sample_device_admin_description"
            android:permission="android.permission.BIND_DEVICE_ADMIN">
            <meta-data android:name="android.app.device_admin" android:resource="@xml/my_admin" />
            <intent-filter>
                <action android:name="android.app.action.DEVICE_ADMIN_ENABLED" />
            </intent-filter>
</receiver>
<uses-permission android:name="android.permission.MANAGE_DEVICE_ADMINS" />
{% endhighlight %}

The following line of meta-data need to match a xml resource file, in this case ```my_admin```

```  <meta-data android:name="android.app.device_admin" android:resource="@xml/my_admin" />```

The content of ```my_admin.xml```
{% highlight xml  %}
<device-admin xmlns:android="http://schemas.android.com/apk/res/android">
    <uses-policies>
        <limit-password />
        <watch-login />
        <reset-password />
        <force-lock />
        <wipe-data />
    </uses-policies>
</device-admin>
{% endhighlight %}



#### Set your app as device owner
Now you can set your app as deivce own with dpm command under adb shell.

{% highlight java  %}
adb shell
dpm set-device-owner wenchao.kiosk/.MyAdmin
{% endhighlight %}

Replace wenchao.kiosk with your package name. ```/.MyAdmin``` is your receiver class, if your receiver class is in different package of you app package, specify full path such as ```wenchao.kiosk/other.package.name.MyAdmin ```

Now, before you call ```startLockTask()```, you need to allow the your app to pin the screen as device owner


{% highlight java %}
@Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        // Set the app into full screen mode
        getWindow().getDecorView().setSystemUiVisibility(flags);

        //Following code allow the app packages to lock task in true kiosk mode
        setContentView(wenchao.kiosk.R.layout.activity_lock_activity);
        // get policy manager
        DevicePolicyManager myDevicePolicyManager = (DevicePolicyManager) getSystemService(Context.DEVICE_POLICY_SERVICE);
        // get this app package name
        ComponentName mDPM = new ComponentName(this, MyAdmin.class);

        if (myDevicePolicyManager.isDeviceOwnerApp(this.getPackageName())) {
            // get this app package name
            String[] packages = {this.getPackageName()};
            // mDPM is the admin package, and allow the specified packages to lock task
            myDevicePolicyManager.setLockTaskPackages(mDPM, packages);
            startLockTask();
        } else {
            Toast.makeText(getApplicationContext(),"Not owner", Toast.LENGTH_LONG).show();
        }
      }
{% endhighlight %}

Now, the home button in navigation disappear and user have no way to go out of your app. The back button is there, but it will do nothing
<figure>
  <img src="/images/kiosk-lock.png" alt="image">
</figure>


### Boot into your kiosk app
Specify your activity as a HOME intent by modifying intent-filter of your activity in ```Manifest.xml```

{% highlight xml  %}
<intent-filter>
   <action android:name="android.intent.action.MAIN" />
   <category android:name="android.intent.category.HOME" />
   <category android:name="android.intent.category.DEFAULT" />
</intent-filter>
{% endhighlight %}



Then, you need to manually replace default system launcher with your app by going to ```Settings > Home``` , and choice your app as the home app. These only needs to be down once.

Once done, if you reboot the system, you app will be booted up and pin the screen. User have not no way to access the other part of your system. Note, if your system crash, the default andriod lanucher can kick in.


### Full screen
Visible navigation bars are not ideal for my project, so I decided to hide it. You can do it by setting some system window flags.

{% highlight java %}
final int flags = View.SYSTEM_UI_FLAG_LAYOUT_STABLE
            | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION
            | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
            | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION
            | View.SYSTEM_UI_FLAG_FULLSCREEN
            | View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        /* Set the app into full screen mode */
        getWindow().getDecorView().setSystemUiVisibility(flags);
        //leave out other code ...

    }

{% endhighlight %}

With my Approach, a transient navigation bar will still be visible when user swipe on the screen from top or button of the screen. It will disappear after 1-2 seconds.

<figure>
  <img src="/images/kiosk-fullscreen.png" alt="image">
</figure>



### Keep device awake
This can be simply done by putting a line in ```Manifest.xml```, under your activity.
```android:keepScreenOn="true"```


### Disable volume buttons
This can be down by trapping volume up and down event.

{% highlight java %}
@Override
    public boolean onKeyDown(int keyCode, KeyEvent event){

        if (keyCode == KeyEvent.KEYCODE_VOLUME_UP){
            Toast.makeText(this, "Volume button is disabled", Toast.LENGTH_SHORT).show();
            return true;
        }

        if (keyCode == KeyEvent.KEYCODE_VOLUME_DOWN){
            Toast.makeText(this, "Volume button is disabled", Toast.LENGTH_SHORT).show();
            return true;
        }

        return super.onKeyDown(keyCode, event);
    }
{% endhighlight %}

You programmatically set the volumes to max or mute as well

{% highlight java %}
private void setVolumMax(){
        AudioManager am = (AudioManager) getSystemService(Context.AUDIO_SERVICE);
        am.setStreamVolume(
                AudioManager.STREAM_SYSTEM,
                am.getStreamMaxVolume(AudioManager.STREAM_SYSTEM),
                0);
    }
{% endhighlight %}

### Disable power button
Ideally I want to do this, but it seems not easy without modification of firmwares.
