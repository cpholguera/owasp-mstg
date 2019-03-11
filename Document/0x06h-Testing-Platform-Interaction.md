## iOS Platform APIs

### Testing App Permissions

#### Overview

iOS makes all mobile applications run under the `mobile` user. Each application is sandboxed and limited using policies enforced by the Trusted BSD mandatory access control framework. These policies are called profiles and all third-party applications use on generic sandbox profile: the container permission list. See the [archived Apple Developer Documentation](https://developer.apple.com/library/archive/documentation/Security/Conceptual/AppSandboxDesignGuide/AppSandboxInDepth/AppSandboxInDepth.html "Apple Developer Documentation on Sandboxing") and the [newer Apple Developer Security Documentation](https://developer.apple.com/documentation/security "Apple Developer Security Documentation") for more details.

##### Data and Resources Protected by System Authorization Settings

On iOS, apps need to request permission to the user for accessing data and resources protected by system authorization settings:

- Bluetooth peripherals
- Calendar data
- Camera
- Contacts
- Health sharing
- Health updating
- HomeKit
- Location
- Microphone
- Motion
- Music and the media library
- Photos
- Reminders
- Siri
- Speech recognition
- the TV provider

For more details, check the [archived App Programming Guide for iOS](https://developer.apple.com/library/archive/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/ExpectedAppBehaviors/ExpectedAppBehaviors.html#//apple_ref/doc/uid/TP40007072-CH3-SW7 "Data and resources protected by system authorization settings") and the article "[Protecting the User's Privacy](https://developer.apple.com/documentation/uikit/core_app/protecting_the_user_s_privacy "Protecting the User's Privacy")" at Apple's Developer Documentation.
Even though Apple urges to protect the privacy of the user and be [very clear on how to ask permissions](https://developer.apple.com/design/human-interface-guidelines/ios/app-architecture/requesting-permission/ "Requesting Permissions"), it can still be the case that an app requests too many permissions.

##### Device Capabilities

An app might have a set of device capabilities (`UIRequiredDeviceCapabilities`), which can be required by the developer in order to run the app. These capabilities are listed at the [Apple Developer Documentation](https://developer.apple.com/library/archive/documentation/General/Reference/InfoPlistKeyReference/Articles/iPhoneOSKeys.html#//apple_ref/doc/plist/info/UIRequiredDeviceCapabilities "UIRequiredDeviceCapabilities") and are used by App Store and by iTunes to ensure that only compatible devices are listed. Many of these capabilities do not require the user to provide permission. Note that the actual available capabilities differ per type of developer profile used to sign the application. See the [Apple Developer Documentation](https://developer.apple.com/support/app-capabilities/ "Advanced App Capabilities") for more details.

##### Entitlements

As stated in the [Apple Developer Documentation](https://developer.apple.com/library/archive/documentation/Miscellaneous/Reference/EntitlementKeyReference/Chapters/AboutEntitlements.html), entitlements confer specific capabilities or security permissions to iOS apps. Many entitlements can be set using the "Summary" tab of the Xcode target editor. Other entitlements require editing a target’s entitlements property list file or are inherited from the iOS provisioning profile used to run the app.

The [Apple Developer Documentation](https://developer.apple.com/library/archive/technotes/tn2415/_index.html#//apple_ref/doc/uid/DTS40016427-CH1-APPENTITLEMENTS) also explains that:

- during code signing, the entitlements corresponding to the app’s enabled Capabilities/Services are transferred to the app's signature from the provisioning profile Xcode chose to sign the app.
- the provisioning profile is embedded into the app bundle during the build (`embedded.mobileprovision`).
- entitlements from Code Signing Entitlements files (`<appname>.entitlements`) are transferred to the app's signature.

For example, if a developer wants to set the "Default Data Protection" capability, he would go to the "Capabilities" tab in Xcode and enable "Data Protection", this is directly written by Xcode to the `<appname>.entitlements` as the `com.apple.developer.default-data-protection` entitlement with default value `NSFileProtectionComplete`. In the IPA we might find this in the `embedded.mobileprovision` as:

```xml
<key>Entitlements</key>
<dict>
    ...
    <key>com.apple.developer.default-data-protection</key>
    <string>NSFileProtectionComplete</string>
</dict>
```

For other capabilities such as HealthKit, the user has to be asked for permission, therefore it is not enough to add the entitlements, special keys and strings have to be added to the `Info.plist` file of the app.

The following sections go more into detail about the mentioned files and how to perform static and dynamic analysis using them.

#### Static Analysis

Since iOS 10, these are the main areas which you need to inspect for permissions:

Since iOS 10, there are three areas which you need to inspect for permissions:
- the Info.plist file,
- the `<appname>.enttitlements` file, where <appname> is the name of the application
- the source-code.

##### Info.plist
The Info.plist contains the texts offered to users when requesting permission to access the protected data or resources. The [Apple Documentation](https://developer.apple.com/design/human-interface-guidelines/ios/app-architecture/requesting-permission/ "Requesting Permission") gives a clear instruction on how the user should be asked for permission to access the given resource. Following these guidelines should make it relatively simple to evaluate each and every entry in the Info.plist file to check if the permission makes sense.
For example, when you have a Info.plist file, for a Solitair game which has, at least, the following content:

```xml
<key>NSHealthClinicalHealthRecordsShareUsageDescription</key>
<string>Share your health data with us!</string>
<key>NSCameraUsageDescription</key>
<string>We want to access your camera</string>
```

It should be suspicious that a regular solitaire game requests this kind of resource access as it probably does not have any need for [accessing the camera](https://developer.apple.com/library/archive/documentation/General/Reference/InfoPlistKeyReference/Articles/CocoaKeys.html#//apple_ref/doc/uid/TP40009251-SW24) nor a [user's health-records](https://developer.apple.com/library/archive/documentation/General/Reference/InfoPlistKeyReference/Articles/CocoaKeys.html#//apple_ref/doc/uid/TP40009251-SW76).


##### Code Signing Entitlements File

Certain capabilities require a [code signing entitlements file](https://developer.apple.com/library/archive/technotes/tn2415/_index.html#//apple_ref/doc/uid/DTS40016427-CH1-ENTITLEMENTSFILE) (`<appname>.entitlements`). It is automatically generated by Xcode but may be also manually edited and/or extended by the developer.

Here is an example of entitlements file of the [open source app Telegram](https://github.com/peter-iakovlev/Telegram-iOS/blob/77ee5c4dabdd6eb5f1e2ff76219edf7e18b45c00/Telegram-iOS/Telegram-iOS-AppStoreLLC.entitlements#L23) including the [App Groups entitlement](https://developer.apple.com/documentation/foundation/com_apple_security_application-groups "App Groups entitlement") (`application-groups`):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
...
    <key>com.apple.security.application-groups</key>
    <array>
        <string>group.ph.telegra.Telegraph</string>
    </array>
</dict>
...
</plist>
```

When using some entitlements like this one, no additional permissions provided by the user are required. Therefore, it is important to properly verify them as the app could be potentially leaking information to other apps if this entitlement is not properly configured.

As documented at [Apple Developer Documentation](https://developer.apple.com/library/archive/documentation/Miscellaneous/Reference/EntitlementKeyReference/Chapters/EnablingAppSandbox.html#//apple_ref/doc/uid/TP40011195-CH4-SW19 "Adding an App to an App Group"), the App Groups entitlement is required to share information between different apps through IPC or a shared file container, which means that data can be shared on the device directly between the apps.
This entitlement is also required if an app extension requires to [share information with its containing app](https://developer.apple.com/library/archive/documentation/General/Conceptual/ExtensibilityPG/ExtensionScenarios.html "Sharing Data with Your Containing App").

Depending on the data to-be-shared it might be more appropriate to share it using another method such as through a back end were this data could be potentially verified, avoiding tampering by e.g. the user himself.

##### Embedded Provisioning Profile File

If not having the original source code project you should then analyze the IPA and search inside for the *embedded provisioning profile* that is usually located in the root app bundle folder (`Payload/<appname>.app/`) under the name `embedded.mobileprovision`.

This file is not a `.plist`, it is encoded using [Cryptographic Message Syntax](https://en.wikipedia.org/wiki/Cryptographic_Message_Syntax). On macOS you can [inspect an embedded provisioning profile's entitlements](https://developer.apple.com/library/archive/technotes/tn2415/_index.html#//apple_ref/doc/uid/DTS40016427-CH1-PROFILESENTITLEMENTS) using the following command:

```bash
$ security cms -D -i embedded.mobileprovision
```

and then search for the Entitlements key region (`<key>Entitlements</key>`).

##### Entitlements Embedded in the Compiled App Binary

If you only have the compiled app (IPA) or even only the running app, you normally won't be able to find `.entitlements` files. This could be also the case for the `embedded.mobileprovision` file. Still, you'll be able to extract the entitlements files from the app bundle yourself.

First you need to find the path to the app's bundle. This can be easily done with objection, example using Telegram:

- open the app and leave it running on foreground
- run the following command

```bash
$ objection --gadget Telegram run env | grep BundlePath
BundlePath  /var/containers/Bundle/Application/15E6A58F-1CA7-44A4-A9E0-6CA85B65FA35/Telegram X.app
```

Now you have two options:

- connect per SSH, `cd` to the bundle and locate the (encrypted) app binary and use grep directly on it
- or decrypt and extract the binary to your computer and use other tools apart from grep like binwalk or radare2.

> Note: don't rely on the `strings` command for this kind of things as it won't be able to find this information. Better use grep with the `-a` flag directly on the binary or use radare2 (`izz`)/rabin2 (`-zz`).

Now let's assume that you have the binary on your computer. For Telegram it's called "Telegram X".

One approach is to use binwalk to extract (`-e`) all XML files (`-y=xml`):

```language
$ binwalk -e -y=xml ./Telegram\ X

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
1430180       0x15D2A4        XML document, version: "1.0"
1458814       0x16427E        XML document, version: "1.0"
```

Or you can use radare2 (`-qc to *quietly* run one command and exit`) to search all strings on the app binary (`izz`) containing "PropertyList" (`~PropertyList`):

```bash
$ r2 -qc 'izz~PropertyList' ./Telegram\ X

24162 0x0015d2a4 0x0015d2a4 1526 1527 () ascii <?xml version="1.0" encoding="UTF-8" standalone="yes"?>\n<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">\n<plist version="1.0">
...<key>com.apple.security.application-groups</key>\n\t\t<array>\n\t\t\t<string>group.ph.telegra.Telegraph</string>...

24696 0x0016427d 0x0016427d 331 332 () ascii H<?xml version="1.0" encoding="UTF-8"?>\n<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">\n<plist version="1.0">\n<dict>\n\t<key>cdhashes</key>...
```

In both cases (binwalk or radare2) we were able to extract the same two `plist` files. If we inspect the first one (0x0015d2a4) we see that we were able to completely recover the [original entitlements file from Telegram](https://github.com/peter-iakovlev/Telegram-iOS/blob/77ee5c4dabdd6eb5f1e2ff76219edf7e18b45c00/Telegram-iOS/Telegram-iOS-AppStoreLLC.entitlements).


##### Source Code Inspection

After having checked the `<appname>.entitlements` file and the `Info.plist` file, it is time to verify how the requested permissions and assigned capabilities are put to use. For this, a source code review should be enough.

Pay attention to:

- whether the *purpose strings* in the `Info.plist` file match the programmatic implementations.
- whether the capabilities registered are used in such a way that no confidential information is leaking.

Note that apps should crash if a capability is required to use which requires a permission without the permission-explanation-text being registered at the Info.plist file.

#### Dynamic Analysis
There are various steps in the analysis process:
- Check the embedded.mobileprovision file and the <appname>.entitlements file and see which capbilities they contain.
- Obtain the Info.plist file and check for which permissions it provided an explanation.
- Go through the application and check whether the application communicates with other applications or with back-ends. Check whether the information retrieved using the permissions and capabilities are used for ill-purposed or are over-asked/under-utilized.

You can now keep going and try to trace and verify how the data is being validated. For example, if you have two apps that *communicate* via Universal Links you can use this to see if the sending app is leaking sensitive data by hooking these methods in the receiving app. This is especially useful when you don't have the source code as you will be able to retrieve the full URL that you wouldn't see other way as it might be the result of clicking some button or triggering some functionality.

In some cases, you might find data in `userInfo` of the `NSUserActivity` object. In the previous case there was no data being transferred but it might be the case for other cases. To see this, be sure to hook the `userInfo` property or access it directly form the `continueUserActivity` object in your hook (e.g. by adding a line like this `log("userInfo:" + ObjC.Object(args[3]).userInfo().toString());`).

**Final Notes about Universal Links and Handoff**

A final note here, this is also [the way Handoff works](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/Handoff/HandoffFundamentals/HandoffFundamentals.html#//apple_ref/doc/uid/TP40014338) and how you could also see the received data. Actually, this is very similar to ["Web Browser–to–Native App Handoff"](https://developer.apple.com/library/archive/documentation/UserExperience/Conceptual/Handoff/AdoptingHandoff/AdoptingHandoff.html#//apple_ref/doc/uid/TP40014338-CH2-SW10) scenario.

> If the user is using a web browser on the originating device, and the receiving device is an iOS device with a native app that claims the domain portion of the `webpageURL` property, then iOS launches the native app and sends it an `NSUserActivity` object with an `activityType` value of `NSUserActivityTypeBrowsingWeb`. The `webpageURL` property contains the URL the user was visiting, while the `userInfo` dictionary is empty.

Like Universal Links, the Handoff Activity Continuation is defined in the `com.apple.developer.associated-domains` entitlement and in the `apple-app-site-association` file from the server, both using the keyword `"activitycontinuation":`. See ["Retrieving the App Site Association File"](#Retrieving-the-App-Site-Association-File) above for an example.


#### UIActivity Sharing

##### Overview

Starting on iOS 6 it is possible for third-party apps to share information via specific mechanisms [like AirDrop, for example](https://developer.apple.com/library/archive/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/Inter-AppCommunication/Inter-AppCommunication.html#//apple_ref/doc/uid/TP40007072-CH6-SW3 "Supporting AirDrop"). From a user perspective, this feature is the well-known *share activity sheet* that appears after clicking on the "Share" button.

![Share Activity Sheet](Images/Chapters/0x06h/share_activity_sheet_1.png)

The available sharing mechanisms (aka. Activity Types) include:

- airDrop
- assignToContact
- copyToPasteboard
- mail
- message
- postToFacebook
- postToTwitter

A full list can be found in [UIActivity.ActivityType](https://developer.apple.com/documentation/uikit/uiactivity/activitytype). If not considered appropriate for the app, the developers have the possibility to exclude some of these sharing mechanisms.

##### Static Analysis

###### Sending Files

When testing `UIActivity` Sharing you should pay especial attention to:

- the data being shared,
- the custom activities,
- the excluded activity types.
  
Data sharing via `UIActivity` works by creating a `UIActivityViewController` and passing it the desired data (URLs, text, a picture) on [`init(activityItems:applicationActivities:)`](https://developer.apple.com/documentation/uikit/uiactivityviewcontroller/1622019-init).

As we mentioned before, it is possible to exclude some of the sharing mechanisms via the controller's [`excludedActivityTypes` property](https://developer.apple.com/documentation/uikit/uiactivityviewcontroller/1622009-excludedactivitytypes). It is highly recommended to do the tests using the latest versions of iOS as the number of activity types that can be excluded can increase and the developers have to be aware of this and **explicitely exclude** the ones that are not appropriate for the app data. Some might not be even documented like "Create Watch Face".

If having the source code, you should take a look at the `UIActivityViewController` and:

- inspect the activities passed to the `init(activityItems:applicationActivities:)` method
- check if it defines custom activities (also being passed to the previous method)
- verify the `excludedActivityTypes` if any

If you only have the compiled/installed app, try searching for the previous method and property, for example:

```bash
$ rabin2 -zq Telegram\ X.app/Telegram\ X | grep -i activityItems
0x1000df034 45 44 initWithActivityItems:applicationActivities:
```

###### Receiving Files

When receiving files, you should check:

- if the app declares *custom document types* by looking into Exported/Imported UTIs ("Info" tab of the Xcode project). Here is the list of all [system declared UTIs](https://developer.apple.com/library/archive/documentation/Miscellaneous/Reference/UTIRef/Articles/System-DeclaredUniformTypeIdentifiers.html#//apple_ref/doc/uid/TP40009259) (Uniform Type Identifiers).
- if the app specifies any *document types that it can open* by looking into Document Types ("Info" tab of the Xcode project). If present, they consist of name and one or more UTIs that represent the data type (e.g. "public.png" for PNG files). iOS uses this to determine if the app is eligible to open a given document (specifying Exported/Imported UTIs is not enough).
- if the app properly *verifies the received data* by looking into the implementation of [`application:openURL:options:`](https://developer.apple.com/documentation/uikit/uiapplicationdelegate/1623112-application?language=objc) (or its deprecated version [`application:openURL:sourceApplication:annotation:`](https://developer.apple.com/documentation/uikit/uiapplicationdelegate/1623073-application?language=objc)) in the app delegate.

If not having the source code you can still take a look into the `Info.plist` file and search for:

- `UTExportedTypeDeclarations`/`UTImportedTypeDeclarations` if the app declares exported/imported *custom document types*.
- `CFBundleDocumentTypes` to see if the app specifies any *document types that it can open*.

A very complete explanation about the use of these key can be found [here](https://stackoverflow.com/questions/21937978/what-are-utimportedtypedeclarations-and-utexportedtypedeclarations-used-for-on-i).

Let's see a real-world example. We will take a File Manager app and take a look at these keys. We used [objection](https://github.com/sensepost/objection) here to read the `Info.plist` file.

```bash
objection --gadget SomeFileManager run ios plist cat Info.plist
```

> Note that this is the same as if we would retrieve the IPA from the phone or accessed via e.g. SSH and navigated to the corresponding folder in the IPA / app sandbox. However, with objection we are just *one command away* from our goal and this can be still considered static analysis.

The first thing we noticed is that app does not declare any imported custom document types but ye could find a couple of exported ones:

```xml
UTExportedTypeDeclarations =     (
            {
        UTTypeConformsTo =             (
            "public.data"
        );
        UTTypeDescription = "SomeFileManager Files";
        UTTypeIdentifier = "com.some.filemanager.custom";
        UTTypeTagSpecification =             {
            "public.filename-extension" =                 (
                ipa,
                deb,
                zip,
                rar,
                tar,
                gz,
                ...
                key,
                pem,
                p12,
                cer
            );
        };
    }
);
```

The app also declares the document types it opens as we can find the key `CFBundleDocumentTypes`:

```xml
CFBundleDocumentTypes =     (
        {
        ...
        CFBundleTypeName = "SomeFileManager Files";
        LSItemContentTypes =             (
            "public.content",
            "public.data",
            "public.archive",
            "public.item",
            "public.database",
            "public.calendar-event",
            ...
        );
    }
);
```

We can see that this File Manager will try to open anything that conforms to any of the UTIs listed in `LSItemContentTypes` and it's ready to open files with the extensions listed in `UTTypeTagSpecification/"public.filename-extension"`. Please take a note of this because it will be useful if you want to search for vulnerabilities when dealing with the different types of files when performing dynamic analysis.

##### Dynamic Analysis

###### Sending Files

There are three main things you can easily inspect by performing dynamic instrumentation:

- the `activityItems`: an array of the items being shared, that might be of different types, e.g. one string and one picture to be shared via a messaging app
- the `applicationActivities`: an array of `UIActivity` objects representing the app's custom services
- the `excludedActivityTypes`: an array of the Activity Types that are not supported, e.g. `postToFacebook`

To achieve this you can do two things:

- hook the method we have seen in the static analysis, [`init(activityItems:applicationActivities:)`](https://developer.apple.com/documentation/uikit/uiactivityviewcontroller/1622019-init) to get the `activityItems` and `applicationActivities`
- find out the excluded activities by hooking [`excludedActivityTypes` property](https://developer.apple.com/documentation/uikit/uiactivityviewcontroller/1622009-excludedactivitytypes)

Let's see an example using Telegram to share a picture and a text file. First prepare the hooks, we will use the Frida REPL and write a script for this:

```javascript
Interceptor.attach(
ObjC.classes.
    UIActivityViewController['- initWithActivityItems:applicationActivities:'].implementation, {
  onEnter: function (args) {

    printHeader(args)

    this.initWithActivityItems = ObjC.Object(args[2]);
    this.applicationActivities = ObjC.Object(args[3]);

    console.log("initWithActivityItems: " + this.initWithActivityItems);
    console.log("applicationActivities: " + this.applicationActivities);

  },
  onLeave: function (retval) {
    printRet(retval);
  }
});

Interceptor.attach(
ObjC.classes.UIActivityViewController['- excludedActivityTypes'].implementation, {
  onEnter: function (args) {
    printHeader(args)
  },
  onLeave: function (retval) {
    printRet(retval);
  }
});

function printHeader(args) {
  console.log(Memory.readUtf8String(args[1]) + " @ " + args[1])
};

function printRet(retval) {
  console.log('RET @ ' + retval + ': ' );
  try {
    console.log(new ObjC.Object(retval).toString());
  } catch (e) {
    console.log(retval.toString());
  }
};
```

You can store this as a JavaScript file, e.g. `inspect_send_activity_data.js` and load it like this:

```bash
$ frida -U Telegram -l inspect_send_activity_data.js
```

Now observe the output when you first share a picture:

```javascript
[*] initWithActivityItems:applicationActivities: @ 0x18c130c07
initWithActivityItems: (
    "<UIImage: 0x1c4aa0b40> size {571, 264} orientation 0 scale 1.000000"
)
applicationActivities: nil
RET @ 0x13cb2b800:
<UIActivityViewController: 0x13cb2b800>

[*] excludedActivityTypes @ 0x18c0f8429
RET @ 0x0:
nil
```

and then a text file:

```javascript
[*] initWithActivityItems:applicationActivities: @ 0x18c130c07
initWithActivityItems: (
    "<QLActivityItemProvider: 0x1c4a30140>",
    "<UIPrintInfo: 0x1c0699a50>"
)
applicationActivities: (
)
RET @ 0x13c4bdc00:
<_UIDICActivityViewController: 0x13c4bdc00>

[*] excludedActivityTypes @ 0x18c0f8429
RET @ 0x1c001b1d0:
(
    "com.apple.UIKit.activity.MarkupAsPDF"
)
```

You can see that,

- for the picture, the activity item is a `UIImage` and there are no excluded activities.
- for the text file there are two different activity items and "com.apple.UIKit.activity.MarkupAsPDF" is excluded.

In the previous example, there were no custom `applicationActivities` and only one excluded activity. However, to better illustrate what you can expect from other apps we have shared a picture using another app, here you can see a bunch of application activities and excluded activities (output was edited to hide the name of the originating app):

```javascript
[*] initWithActivityItems:applicationActivities: @ 0x18c130c07
initWithActivityItems: (
    "<SomeActivityItemProvider: 0x1c04bd580>"
)
applicationActivities: (
    "<SomeActionItemActivityAdapter: 0x141de83b0>",
    "<SomeActionItemActivityAdapter: 0x147971cf0>",
    "<SomeOpenInSafariActivity: 0x1479f0030>",
    "<SomeOpenInChromeActivity: 0x1c0c8a500>"
)
RET @ 0x142138a00:
<SomeActivityViewController: 0x142138a00>

[*] excludedActivityTypes @ 0x18c0f8429
RET @ 0x14797c3e0:
(
    "com.apple.UIKit.activity.Print",
    "com.apple.UIKit.activity.AssignToContact",
    "com.apple.UIKit.activity.SaveToCameraRoll",
    "com.apple.UIKit.activity.CopyToPasteboard",
)
```

###### Receiving Files

After performing the static analysis you would know which *document types that the app can open* and *if it declares any custom document types* and (part of) the methods involved. You can use this now to test the receiving part by:

- sending a file that will trigger the "Open with..." dialogue (via AirDrop or e-mail) or *share* a file with the app from another app
- hooking `application:openURL:options:` and any other methods that we have identified in a previous static analysis
- observing the app behaviour
- in addition, you could send specific malformed files and/or use a fuzzing technique

To illustrate this with an example we have chosen the same real-world file manager app from the static analysis section and do the following:

1. send a PDF file from our MacBook via Airdrop
2. the "AirDrop" popup appears
3. if we accept, as there is no default app that will open the file, it switches to the "Open with..." popup
4. we can select the app that will open our file. The next screenshot shows this (we have modified the display name using Frida to conceal the app's real name):
    ![AirDrop Open With Dialog](Images/Chapters/0x06h/airdrop_openwith.png)

5. After selecting "SomeFileManager" we can see the following:
    ```javascript
    (0x1c4077000)  -[AppDelegate application:openURL:options:]
    application: <UIApplication: 0x101c00950>
    openURL: file:///var/mobile/Library/Application%20Support
                        /Containers/com.some.filemanager/Documents/Inbox/OWASP_MASVS.pdf
    options: {
        UIApplicationOpenURLOptionsAnnotationKey =     {
            LSMoveDocumentOnOpen = 1;
        };
        UIApplicationOpenURLOptionsOpenInPlaceKey = 0;
        UIApplicationOpenURLOptionsSourceApplicationKey = "com.apple.sharingd";
        "_UIApplicationOpenURLOptionsSourceProcessHandleKey" = "<FBSProcessHandle: 0x1c3a63140; 
                                                                    sharingd:605; valid: YES>";
    }
    0x18c7930d8 UIKit!__58-[UIApplication _applicationOpenURLAction:payload:origin:]_block_invoke
    ...
    0x1857cdc34 FrontBoardServices!-[FBSSerialQueue _performNextFromRunLoopSource]
    RET: 0x1
    ```

As you can see, the sending application is `com.apple.sharingd` and the URL's scheme is `file://`. Note that once we select the app that should open the file, the system already moved the file to the corresponding destination, that is to the app's Inbox. The apps are then responsible for deleting the files inside their Inboxes. This app, for example, moves the file to `/var/mobile/Documents/` and removes it from the Inbox.

```javascript
(0x1c002c760)  -[XXFileManager moveItemAtPath:toPath:error:]
moveItemAtPath: /var/mobile/Library/Application Support/Containers
                            /com.some.filemanager/Documents/Inbox/OWASP_MASVS.pdf
toPath: /var/mobile/Documents/OWASP_MASVS (1).pdf
error: 0x16f095bf8
0x100f24e90 SomeFileManager!-[AppDelegate __handleOpenURL:]
0x100f25198 SomeFileManager!-[AppDelegate application:openURL:options:]
0x18c7930d8 UIKit!__58-[UIApplication _applicationOpenURLAction:payload:origin:]_block_invoke
...
0x1857cd9f4 FrontBoardServices!__FBSSERIALQUEUE_IS_CALLING_OUT_TO_A_BLOCK__
RET: 0x1
```

If you look at the stack trace, you can see how `application:openURL:options:` called `__handleOpenURL:`, which called `moveItemAtPath:toPath:error:`. Notice that we have now this information without having the source code for the target app. The first thing that we had to do was clear: hook `application:openURL:options:`. Regarding the rest, we had to think a little bit and come up with methods that we could start tracing and are related to the file manager, for example, all methods containing the strings "copy", "move", "remove", etc. until we have found that the one being called was `moveItemAtPath:toPath:error:`.

A final thing worth noticing here is that this way of handling incoming files is the same for custom URL schemes. Please refer to ["Testing Custom URL Schemes"](#Testing-Custom-URL-Schemes) for more information.


#### App Extensions

##### Overview

###### What are app extensions?

According to [Apple App Extension Programming Guide](https://developer.apple.com/library/archive/documentation/General/Conceptual/ExtensibilityPG/index.html#//apple_ref/doc/uid/TP40014214-CH20-SW1), app extensions let apps offer custom functionality and content to users while they’re interacting with other apps or the system. In order to do this, they implement specific, well scoped tasks like, for example, define what happens after the user clicks on the "Share" button and selects some app or action, provide the content for a Today widget or enable a custom keyboard.

Depending on the task, the app extension will have a particular type (and only one), the so-called extension points. Some notable ones are:

- Custom Keyboard: Replace the iOS system keyboard with a custom keyboard for use in all apps.
- Share: Post to a sharing website or share content with others.
- Today: also called widgets, they offer content or perform quick tasks in the Today view of Notification Center.

###### How do app extensions interact with other apps?

There are three important elements here:

- App extension: is the one bundled inside a containing app. Host apps interact with it.
- Host app: is the (third-party) app that triggers the app extension of another app.
- Containing app: is the app that contains the app extension bundled into it.

For example, the user selects text in the *host app*, clicks on the "Share" button and selects one "app" or action from the list. This triggers the *app extension* of the *containing app*. The app extension displays its view within the context of the host app and uses the items provided by the host app, the selected text in this case, to perform a specific task (post it on a social network, for example). See this picture from the [Apple App Extension Programming Guide](https://developer.apple.com/library/archive/documentation/General/Conceptual/ExtensibilityPG/ExtensionOverview.html#//apple_ref/doc/uid/TP40014214-CH2-SW13) which pretty good summarizes this:

![App Extensions Communication](Images/Chapters/0x06h/app_extensions_communication.png)

###### Security Considerations

From the security point of view it is important to note that

- an app extension does never communicate directly with its containing app (typically, it isn’t even running while the contained app extension is running).
- an app extension and the host app communicate via inter-process communication.
- an app extension’s containing app and the host app don’t communicate at all.
- a Today widget (and no other app extension type) can ask the system to open its containing app by calling the `openURL:completionHandler:` method of the `NSExtensionContext` class.
- any app extension and its containing app can access shared data in a privately defined shared container.

In addition:

- app extensions cannot access some APIS, for example, HealthKit.
- they cannot receive data using AirDrop but do can send data.
- no long-running background tasks are allowed but uploads or downloads can be initiated.
- app extensions cannot access the camera or microphone on an iOS device (except for iMessage app extensions).

###### Things to consider when testing App Extensions

So when we analyze an app we will want to do the following:

- find out if the app integrates app extensions and list them
- verify the data being shared between the app extension and the host app
- check if there are any shared resources between the app extension and the containing app (mostly regarding data coming from the host app)
- verify if the app restricts the use of some app extension

##### Static Analysis

As we anticipated on the overview, the static analysis will take care of:

- [Verifying if the App Contains App Extensions](#Verifying-if-the-App-Contains-App-Extensions)
- [Determining the Supported Data Types](#Determining-the-Supported-Data-Types)
- [Checking Data Sharing with the Containing App](#Checking-Data-Sharing-with-the-Containing-App)
- [Verifying if the App Restricts the Use of App Extensions](#Verifying-if-the-App-Restricts-the-Use-of-App-Extensions)

###### Verifying if the App Contains App Extensions

If you have the original source code you can search for all occurrences of `NSExtensionPointIdentifier` with Xcode (cmd+shift+f) or take a look into "Build Phases / Embed App extensions":

![List App Extensions in Xcode](Images/Chapters/0x06h/xcode_embed_app_extensions.png)

There you can find the names of all embedded app extensions followed by `.appex`, now you can navigate to the individual app extensions in the project.

If not having the original source code:

Grep for `NSExtensionPointIdentifier` among all property lists inside the app bundle.

You can also access per SSH, find the app bundle and list all inside PlugIns or do it with objection:

```
ph.telegra.Telegraph on (iPhone: 11.1.2) [usb] # cd PlugIns
/var/containers/Bundle/Application/15E6A58F-1CA7-44A4-A9E0-6CA85B65FA35/Telegram X.app/PlugIns
ph.telegra.Telegraph on (iPhone: 11.1.2) [usb] # ls
NSFileType      Perms  NSFileProtection    Read    Write    Owner           Group           Size     Creation                   Name
------------  -------  ------------------  ------  -------  --------------  --------------  -------  -------------------------  -------------------------
Directory         493  None                True    False    _installd (33)  _installd (33)  224.0 B  2019-01-31 00:26:06 +0000  NotificationContent.appex
Directory         493  None                True    False    _installd (33)  _installd (33)  512.0 B  2019-01-31 00:26:32 +0000  Widget.appex
Directory         493  None                True    False    _installd (33)  _installd (33)  224.0 B  2019-01-31 00:26:21 +0000  Share.appex
Directory         493  None                True    False    _installd (33)  _installd (33)  192.0 B  2019-01-31 00:26:17 +0000  SiriIntents.appex
```

We can see now the same four app extensions that we saw in Xcode before.

###### Determining the Supported Data Types

This is important for data being shared with host apps (e.g. via Share or Action Extensions). When the user selects some data type in a host app and it matches the data types define here, the host app will offer the extension. It is worth noticing the difference between this and data sharing via `UIActivity` where we had to define the document types, also using UTIs. An app does not need to have an extension or that. It is possible to share data using only `UIActivity`.

Inspect the app extension's `Info.plist` file and search for `NSExtensionActivationRule`. That key specifies the data being supported as well as e.g. maximum of items supported. For example:

```xml
<key>NSExtensionAttributes</key>
    <dict>
        <key>NSExtensionActivationRule</key>
        <dict>
            <key>NSExtensionActivationSupportsImageWithMaxCount</key>
            <integer>10</integer>
            <key>NSExtensionActivationSupportsMovieWithMaxCount</key>
            <integer>1</integer>
            <key>NSExtensionActivationSupportsWebURLWithMaxCount</key>
            <integer>1</integer>
        </dict>
    </dict>
```

Only the data types present here and not having `0` as `MaxCount` will be supported. However, more complex filtering is possible by using a so-called predicate string that will evaluate the UTIs given. Please refer to the [Apple App Extension Programming Guide](https://developer.apple.com/library/archive/documentation/General/Conceptual/ExtensibilityPG/ExtensionScenarios.html#//apple_ref/doc/uid/TP40014214-CH21-SW8) for more detailed information about this.

###### Checking Data Sharing with the Containing App

Remember that app extensions and their containing apps do not have direct access to each other’s containers. However, data sharing can be enabled. This is done via ["App Groups"](https://developer.apple.com/library/archive/documentation/Miscellaneous/Reference/EntitlementKeyReference/Chapters/EnablingAppSandbox.html#//apple_ref/doc/uid/TP40011195-CH4-SW19) and the [`NSUserDefaults`](https://developer.apple.com/documentation/foundation/nsuserdefaults) API. See this figure from [Apple App Extension Programming Guide](https://developer.apple.com/library/archive/documentation/General/Conceptual/ExtensibilityPG/ExtensionScenarios.html#//apple_ref/doc/uid/TP40014214-CH21-SW11):

![App Extensions Container Restrictions](Images/Chapters/0x06h/app_extensions_container_restrictions.png)

As also mentioned in the guide, the app must set up a shared container if the app extension uses the `NSURLSession` class to perform a background upload or download, so that both the extension and its containing app can access the transferred data.

###### Verifying if the App Restricts the Use of App Extensions

It is possible to reject a specific type of app extension by using the method [`application:shouldAllowExtensionPointIdentifier:`](https://developer.apple.com/documentation/uikit/uiapplicationdelegate/1623122-application?language=objc). However, it is currently only possible for "custom keyboard" app extensions (and should be verified when testing apps handling sensitive data via the keyboard like e.g. banking apps).

##### Dynamic Analysis

For the dynamic analysis we can do the following to gain knowledge without having the source code:

- [Inspecting the Items Being Shared](#Inspecting-the-Items-Being-Shared)
- [Identifying the App Extensions Involved](#Identifying-the-App-Extensions-Involved)

Following the previous example of Telegram we will now use the "Share" button on a text file to create a note in the Notes app with it:

![Using an App Extension](Images/Chapters/0x06h/telegram_share_extension.png)


###### Inspecting the Items Being Shared

For this we should hook `NSExtensionContext - inputItems` in the data originating app. If we share a text file from a Telegram chat and select the Notes app as target, we see the following output:

```javascript
(0x1c06bb420) NSExtensionContext - inputItems
0x18284355c Foundation!-[NSExtension _itemProviderForPayload:extensionContext:]
0x1828447a4 Foundation!-[NSExtension _loadItemForPayload:contextIdentifier:completionHandler:]
0x182973224 Foundation!__NSXPCCONNECTION_IS_CALLING_OUT_TO_EXPORTED_OBJECT_S3__
0x182971968 Foundation!-[NSXPCConnection _decodeAndInvokeMessageWithEvent:flags:]
0x182748830 Foundation!message_handler
0x181ac27d0 libxpc.dylib!_xpc_connection_call_event_handler
0x181ac0168 libxpc.dylib!_xpc_connection_mach_event
...
RET: (
    "<NSExtensionItem: 0x1c420a540> - userInfo: {
    NSExtensionItemAttachmentsKey =     (
    "<NSItemProvider: 0x1c46b30e0> {types = (\n \"public.plain-text\",\n \"public.file-url\"\n)}"
    );
}"
)
```

Here we can observe that:

- as we anticipated in the overview, this occurred under-the-hood via XPC, concretely it is implemented via a `NSXPCConnection` that uses the `libxpc.dylib` Framework
- the UTIs included in the `NSItemProvider` are `public.plain-text` and `public.file-url`, the latter being included in `NSExtensionActivationRule` from the [`Info.plist` of the "Share Extension" of Telegram](https://github.com/peter-iakovlev/Telegram-iOS/blob/master/Share/Info.plist).

###### Identifying the App Extensions Involved

You can also find out which app extension is taking care of your the requests and responses by hooking `NSExtension - _plugIn`:

We run the same example again:

```
(0x1c0370200) NSExtension - _plugIn
RET: <PKPlugin: 0x1163637f0 ph.telegra.Telegraph.Share(5.3) 5B6DE177-F09B-47DA-90CD-34D73121C785 1(2) /private/var/containers/Bundle/Application/15E6A58F-1CA7-44A4-A9E0-6CA85B65FA35/Telegram X.app/PlugIns/Share.appex>

(0x1c0372300)  -[NSExtension _plugIn]
RET: <PKPlugin: 0x10bff7910 com.apple.mobilenotes.SharingExtension(1.5) 73E4F137-5184-4459-A70A-83F90A1414DC 1(2) /private/var/containers/Bundle/Application/5E267B56-F104-41D0-835B-F1DAB9AE076D/MobileNotes.app/PlugIns/com.apple.mobilenotes.SharingExtension.appex>
```

As you can see there are two app extensions involved:

- `Share.appex` is sending the text file (`public.plain-text` and `public.file-url`)
- `com.apple.mobilenotes.SharingExtension.appex` which is receiving and will process the text file

If you want to learn more about what's happening under-the-hood in terms of XPC, we recommend to take a look at the internal calls from "libxpc.dylib". For example you can use [`frida-trace`](https://www.frida.re/docs/frida-trace/) and then dig deeper into the methods that you find more interesting by extending the automatically generated stubs.


#### UIPasteboard

##### Overview

The [UIPasteboard](https://developer.apple.com/documentation/uikit/uipasteboard).

##### Static Analysis

##### Dynamic Analysis


### Testing Custom URL Schemes

#### Overview

Custom URL schemes [allow apps to communicate via a custom protocol](https://developer.apple.com/library/content/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/Inter-AppCommunication/Inter-AppCommunication.html#//apple_ref/doc/uid/TP40007072-CH6-SW1 "Using URL Schemes to Communicate with Apps"). An app must declare support for the scheme and handle incoming URLs that use the scheme.

Apple warns about the improper use of custom URL schemes in the [Apple Developer Documentation](https://developer.apple.com/documentation/uikit/core_app/allowing_apps_and_websites_to_link_to_your_content/defining_a_custom_url_scheme_for_your_app):

> URL schemes offer a potential attack vector into your app, so make sure to validate all URL parameters and discard any malformed URLs. In addition, limit the available actions to those that do not risk the user’s data. For example, do not allow other apps to directly delete content or access sensitive information about the user. When testing your URL-handling code, make sure your test cases include improperly formatted URLs.

They also suggest using Universal Links instead, if the purpose is to implement deep linking:

> While custom URL schemes are an acceptable form of deep linking, universal links are strongly recommended as a best practice.

Supporting a custom URL scheme is done by:

- defining the format for the app's URLs
- register the scheme so that the system directs appropriate URLs to the app
- handling the URLs that the app receives.

Security issues arise when an app processes calls to its URL scheme without properly validating the URL and its parameters and when users aren't prompted for confirmation before triggering an important action.

One example is the following [bug in the Skype Mobile app](http://www.dhanjani.com/blog/2010/11/insecure-handling-of-url-schemes-in-apples-ios.html), discovered in 2010: The Skype app registered the `skype://` protocol handler, which allowed other apps to trigger calls to other Skype users and phone numbers. Unfortunately, Skype didn't ask users for permission before placing the calls, so any app could call arbitrary numbers without the user's knowledge. Attackers exploited this vulnerability by putting an invisible `<iframe src="skype://xxx?call"></iframe>` (where `xxx` was replaced by a premium number), so any Skype user who inadvertently visited a malicious website called the premium number.

As a developer, you should carefully validate any URL before calling it. You can whitelist applications which may be opened via the registered protocol handler. Prompting users to confirm the URL-invoked action is another helpful control.

All URLs are passed to the app delegate, either at launch time or while the app is running or in the background. To handle incoming URLs, the delegate should implement methods to:

- retrieve information about the URL and decide whether you want to open it
- open the resource specified by the URL.

More information can be found in the [archived App Programming Guide for iOS](https://developer.apple.com/library/archive/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/Inter-AppCommunication/Inter-AppCommunication.html#//apple_ref/doc/uid/TP40007072-CH6-SW13) and in the [Apple Secure Coding Guide](https://developer.apple.com/library/archive/documentation/Security/Conceptual/SecureCodingGuide/Articles/ValidatingInput.html).

In addition, an app may also want to send URL requests (aka. queries) to other apps. This is done by:

- registering the application query schemes that the app wants to query
- optionally querying other apps to know if they can open a certain URL
- sending the URL requests.

All of this presents a wide attack surface that we will address in the static and dynamic analysis sections.

#### Static Analysis

There are a couple of things that we can do in the static analysis. In the next sections we will see the following:

- [Testing Custom URL Schemes Registration](#Testing-Custom-URL-Schemes-Registration)
- [Testing Application Query Schemes Registration](#Testing-Application-Query-Schemes-Registration)
- [Testing URL Handling and Validation](#Testing-URL-Handling-and-Validation)
- [Testing URL Requests to Other Apps](#Testing-URL-Requests-to-Other-Apps)
- [Testing for Deprecated Methods](#Testing-for-Deprecated-Methods)

##### Testing Custom URL Schemes Registration

The first step to test custom URL schemes is finding out whether an application registers any protocol handlers.

If you have the original source code and want to view registered protocol handlers, simply open the project in Xcode, go to the "Info" tab and open the "URL Types" section as presented in the screenshot below:

![Document Overview](Images/Chapters/0x06h/URL_scheme.png)

Also in Xcode you can find this by searching for the `CFBundleURLTypes` key in the app’s `Info.plist` file (example from [iGoat-Swift](https://github.com/OWASP/iGoat-Swift)):

```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLName</key>
        <string>com.iGoat.myCompany</string>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>iGoat</string>
        </array>
    </dict>
</array>
```

In a compiled application (or IPA), registered protocol handlers are found in the file `Info.plist` in the app bundle's root folder. Open it and search for the `CFBundleURLSchemes` key, if present, it should contain an array of strings (example from [iGoat-Swift](https://github.com/OWASP/iGoat-Swift)):

```xml
grep -A 5 -nri urlsch Info.plist
Info.plist:45:    <key>CFBundleURLSchemes</key>
Info.plist-46-    <array>
Info.plist-47-        <string>iGoat</string>
Info.plist-48-    </array>
```

Once the URL scheme is registered, other apps can open the app that registered the scheme, and pass parameters by creating appropriately formatted URLs and opening them with the [`openURL:options:completionHandler:`](https://developer.apple.com/documentation/uikit/uiapplication/1648685-openurl?language=objc) method.

Note from the [App Programming Guide for iOS](https://developer.apple.com/library/archive/documentation/iPhone/Conceptual/iPhoneOSProgrammingGuide/Inter-AppCommunication/Inter-AppCommunication.html#//apple_ref/doc/uid/TP40007072-CH6-SW7):

> If more than one third-party app registers to handle the same URL scheme, there is currently no process for determining which app will be given that scheme.

This could lead to a URL scheme hijacking attack (see page 136 in [#THIEL]).


##### Testing Application Query Schemes Registration

Before calling the `openURL:options:completionHandler:` method, apps can call [`canOpenURL`](https://developer.apple.com/documentation/uikit/uiapplication/1622952-canopenurl) to verify that the target app is available. However, as this method was being used by malicious app as a way to enumerate installed apps, [from iOS 9.0 the URL schemes passed to it must be also declared](https://developer.apple.com/documentation/uikit/uiapplication/1622952-canopenurl#discussion) by adding the `LSApplicationQueriesSchemes` key to the app's `Info.plist` file and an array of up to 50 URL schemes.

```xml
<key>LSApplicationQueriesSchemes</key>
    <array>
        <string>url_scheme1</string>
        <string>url_scheme2</string>
    </array>
```

`canOpenURL` will always return `NO` for undeclared schemes, whether or not an appropriate app is installed. However, this restriction only applies to `canOpenURL`, **the `openURL:options:completionHandler:` method will still open any URL scheme, even if the `LSApplicationQueriesSchemes` array was declared**, and return `YES` / `NO` depending on the result.

As an example, Telegram declares in its [`Info.plist`](https://github.com/peter-iakovlev/Telegram-iOS/blob/master/Telegram-iOS/Info.plist#L63) these Queries Schemes, among others:

```xml
    <key>LSApplicationQueriesSchemes</key>
    <array>
        <string>dbapi-3</string>
        <string>instagram</string>
        <string>googledrive</string>
        <string>comgooglemaps-x-callback</string>
        <string>foursquare</string>
        <string>here-location</string>
        <string>yandexmaps</string>
        <string>yandexnavi</string>
        <string>comgooglemaps</string>
        <string>youtube</string>
        <string>twitter</string>
        ...
```

##### Testing URL Handling and Validation

In order to determine how a URL path is built and validated, if you have the original source code, you can search for the following methods:

- `application:didFinishLaunchingWithOptions:` method or `application:will-FinishLaunchingWithOptions:`: verify how the decision is made and how the information about the URL is retrieved
- [`application:openURL:options:`](https://developer.apple.com/documentation/uikit/uiapplicationdelegate/1623112-application?language=objc): verify how the resource is being opened, i.e. how the data is being parsed, verify the [options](https://developer.apple.com/documentation/uikit/uiapplication/openurloptionskey), especially if the calling app ([`sourceApplication`](https://developer.apple.com/documentation/uikit/uiapplication/openurloptionskey/1623128-sourceapplication)) is being verified or checked against a white- or blacklist. The app might also need user permission when using the custom URL scheme.

In Telegram you will [find four different methods being used](https://github.com/peter-iakovlev/Telegram-iOS/blob/87e0a33ac438c1d702f2a0b75bf21f26866e346f/Telegram-iOS/AppDelegate.swift#L1250):

```swift
    func application(_ application: UIApplication, open url: URL, sourceApplication: String?) -> Bool {
        self.openUrl(url: url)
        return true
    }
    
    func application(_ application: UIApplication, open url: URL, sourceApplication: String?, annotation: Any) -> Bool {
        self.openUrl(url: url)
        return true
    }
    
    func application(_ app: UIApplication, open url: URL, options: [UIApplicationOpenURLOptionsKey : Any] = [:]) -> Bool {
        self.openUrl(url: url)
        return true
    }
    
    func application(_ application: UIApplication, handleOpen url: URL) -> Bool {
        self.openUrl(url: url)
        return true
    }
```

We can observe some things here:

- the app implements also deprecated methods like [`application:handleOpenURL:`](https://developer.apple.com/documentation/uikit/uiapplicationdelegate/1622964-application?language=objc) and [`application:openURL:sourceApplication:annotation:`](https://developer.apple.com/documentation/uikit/uiapplicationdelegate/1623073-application)
- the source application is not being verified in any of those methods
- all of them call a private `openUrl` method. You can [inspect it](https://github.com/peter-iakovlev/Telegram-iOS/blob/87e0a33ac438c1d702f2a0b75bf21f26866e346f/Telegram-iOS/AppDelegate.swift#L1270) to learn more about how the URL request is handled

##### Testing URL Requests to Other Apps

The method [`openURL:options:completionHandler:`](https://developer.apple.com/documentation/uikit/uiapplication/1648685-openurl?language=objc) and the [deprecated `openURL` method of `UIApplication`](https://developer.apple.com/documentation/uikit/uiapplication/1622961-openurl?language=objc) are responsible for opening URLs (i.e. to send requests / make queries to other apps) that may be local to the current app or it may be one that must be provided by a different app. If you have the original source code you can search directly for usages of those methods.

Additionally, if you are interested into knowing if the app is querying specific services or apps, and if the app is well-known, you can also search for common URL schemes online and include them in your greps. For example, a [quick Google search reveals](https://ios.gadgethacks.com/news/always-updated-list-ios-app-url-scheme-names-0184033/):
    ```
    Apple Music — music:// or musics:// or audio-player-event://
    Calendar — calshow:// or x-apple-calevent://
    Contacts — contacts://
    Diagnostics — diagnostics:// or diags://
    GarageBand — garageband://
    iBooks — ibooks:// or itms-books:// or itms-bookss://
    Mail — message:// or mailto://emailaddress
    Messages — sms://phonenumber
    Notes — mobilenotes://
    ...
    ```.

We search for this method in the telegram source code, this time without using Xcode, just with `egrep`:

```
$ egrep -nr "open.*options.*completionHandler" ./Telegram-iOS/
```

If we inspect the results we will see that this `openURL:options:completionHandler:` is actually for Universal Links so we keep searching for maybe `openURL`:

```
$ egrep -nr "openURL\(" ./Telegram-iOS/
```

If we inspect those lines we will see how this method is also being used to open "Settings" or to open the "App Store Page".

When just searching for `://` we see:

```
if documentUri.hasPrefix("file://"), let path = URL(string: documentUri)?.path {
if !url.hasPrefix("mt-encrypted-file://?") {
guard let dict = TGStringUtils.argumentDictionary(inUrlString: String(url[url.index(url.startIndex, 
    offsetBy: "mt-encrypted-file://?".count)...])) else {
parsedUrl = URL(string: "https://\(url)")
if let url = URL(string: "itms-apps://itunes.apple.com/app/id\(appStoreId)") {
} else if let url = url as? String, url.lowercased().hasPrefix("tg://") {
[[WKExtension sharedExtension] openSystemURL:[NSURL URLWithString:[NSString stringWithFormat:@"tel://%@", userHandle.data]]];
```

After combining the results of both searches and carefully inspecting the source code we find the interesting part for us right now:

```swift
openUrl: { url in
            var parsedUrl = URL(string: url)
            if let parsed = parsedUrl {
                if parsed.scheme == nil || parsed.scheme!.isEmpty {
                    parsedUrl = URL(string: "https://\(url)")
                }
                if parsed.scheme == "tg" {
                    return
                }
            }
            
            if let parsedUrl = parsedUrl {
                UIApplication.shared.openURL(parsedUrl)
```

Before opening a URL, the scheme is validated, "https" will be added if necessary and it won't open any URL with the "tg" scheme. When ready it will use the deprecated `openURL` method.

If only having the compiled application (IPA) you can still try to identify which URL schemes are being used to query other apps.

- check if `LSApplicationQueriesSchemes` was declared or search for common URL schemes
- also use the string `://` or build a regular expression to match URLs as the app might not be declaring some schemes

You can do that by first verifying that the app binary contains those strings by e.g. using unix `strings` command:

```bash
$ strings <yourapp> | grep "someURLscheme://"
```

or even better, use radare2's `iz/izz` command or rafind2, both will find strings where the unix `strings` command won't. Example from iGoat-Swift:

```bash
$ r2 -qc izz~iGoat:// iGoat-Swift
37436 0x001ee610 0x001ee610  23  24 (4.__TEXT.__cstring) ascii iGoat://?contactNumber=
```


##### Testing for Deprecated Methods

Search for deprecated methods like:

- [`application:handleOpenURL:`](https://developer.apple.com/documentation/uikit/uiapplicationdelegate/1622964-application?language=objc)
- [`openURL:`](https://developer.apple.com/documentation/uikit/uiapplication/1622961-openurl?language=objc)
- [`application:openURL:sourceApplication:annotation:`](https://developer.apple.com/documentation/uikit/uiapplicationdelegate/1623073-application)


For example, here we find those three:

```bash
$ rabin2 -zzq Telegram\ X.app/Telegram\ X | grep -i "openurl"

0x1000d9e90 31 30 UIApplicationOpenURLOptionsKey
0x1000dee3f 50 49 application:openURL:sourceApplication:annotation:
0x1000dee71 29 28 application:openURL:options:
0x1000dee8e 27 26 application:handleOpenURL:
0x1000df2c9 9 8 openURL:
0x1000df766 12 11 canOpenURL:
0x1000df772 35 34 openURL:options:completionHandler:
...
```

#### Dynamic Analysis

Once you've identified the custom URL schemes the app has registered, there are several methods that you can use to test them:

- [Performing URL Requests](#Performing-URL-Requests)
- [Identifying and Hooking the URL Handler Method](#Identifying-and-Hooking-the-URL-Handler-Method)
- [Testing URL Schemes Source Validation](#Testing-URL-Schemes-Source-Validation)
- [Fuzzing URL Schemes](#Fuzzing-URL-Schemes)

##### Performing URL Requests

###### Using Safari

To quickly test one URL scheme you can open the URLs on Safari and observe how the app behaves. For example, if you write `tel://123456789` in the address bar of Safari, a pop up will appear with the *telephone number* and the options "Cancel" and "Call". If you press "Call" it will open the Phone app and directly make the call.

You may also know already about pages that trigger custom URL schemes, you can just navigate normally to those pages and Safari will automatically ask when it finds a custom URL scheme.

###### Using Frida

If you simply want to open the URL scheme you can do it using Frida:

```javascript
$ frida -U iGoat-Swift

[iPhone::iGoat-Swift]-> function openURL(url) {
                            var UIApplication = ObjC.classes.UIApplication.sharedApplication();
                            var toOpen = ObjC.classes.NSURL.URLWithString_(url);
                            return UIApplication.openURL_(toOpen);
                        }
[iPhone::iGoat-Swift]-> openURL("tel://234234234")
true
```

Or as in this example from [Frida CodeShare](https://codeshare.frida.re/@dki/ios-url-scheme-fuzzing/) where the author uses the non-public API `LSApplicationWorkspace.openSensitiveURL:withOptions:` to open the URLs (from the SpringBoard app):

```javascript
function openURL(url) {
    var w = ObjC.classes.LSApplicationWorkspace.defaultWorkspace();
    var toOpen = ObjC.classes.NSURL.URLWithString_(url);
    return w.openSensitiveURL_withOptions_(toOpen, null);
}
```

> Note that the use of non-public APIs is not permitted on the App Store, that's why we don't even test these but we are allowed to use them for our dynamic analysis.

###### Using IDB

For this you can also use [IDB](https://www.idbtool.com/):

- Start IDB, connect to your device and select the target app. You can find details in the [IDB documentation](https://www.idbtool.com/documentation/setup.html).
- Go to the "URL Handlers" section. In "URL schemes", click "Refresh", and on the left you'll find a list of all custom schemes defined in the app being tested. You can load these schemes by clicking "Open", on the right side. By simply opening a blank URI scheme (e.g., opening `myURLscheme://`), you can discover hidden functionality (e.g., a debug window) and bypass local authentication.

###### Using Needle

Needle can be used to test custom URL schemes, the following module can be used to open the URLs (URIs):

```
[needle] >
[needle] > use dynamic/ipc/open_uri
[needle][open_uri] > show options

  Name  Current Value  Required  Description
  ----  -------------  --------  -----------
  URI                  yes       URI to launch, eg tel://123456789 or http://www.google.com/

[needle][open_uri] > set URI "myapp://testpayload'"
URI => "myapp://testpayload'"
[needle][open_uri] > run
```

Manual fuzzing can be performed against the URL scheme to identify input validation and memory corruption bugs.

##### Identifying and Hooking the URL Handler Method

If you can't look into the original source code you will have to find out yourself which method does the app use to handle the URL scheme requests that it receives. You cannot know if it is an Objective-C method or a Swift one, or even if the app is using a deprecated one.

###### Crafting the link yourself and letting Safari open it (objc-method-observer)

So let's hook all of them to find out. For this we will use the [ObjC method observer](https://codeshare.frida.re/@mrmacete/objc-method-observer/) from Frida CodeShare, which is an extremely handy script that allows you to quickly observe any collection of methods or classes just by providing a simple pattern.

In this case we are interested into all methods containing "openURL", therefore our pattern will be `*[* *openURL*]`:

- the first asterisk will match all instance `-` and class `+` methods
- the second matches all Objective-C classes
- the third and forth allow to match any method containing the string `openURL`


```javascript
$ frida -U iGoat-Swift --codeshare mrmacete/objc-method-observer

[iPhone::iGoat-Swift]-> observeSomething("*[* *openURL*]");
Observing  -[_UIDICActivityItemProvider activityViewController:openURLAnnotationForActivityType:]
Observing  -[CNQuickActionsManager _openURL:]
Observing  -[SUClientController openURL:]
Observing  -[SUClientController openURL:inClientWithIdentifier:]
Observing  -[FBSSystemService openURL:application:options:clientPort:withResult:]
Observing  -[iGoat_Swift.AppDelegate application:openURL:options:]
Observing  -[PrefsUILinkLabel openURL:]
Observing  -[UIApplication openURL:]
Observing  -[UIApplication _openURL:]
Observing  -[UIApplication openURL:options:completionHandler:]
Observing  -[UIApplication openURL:withCompletionHandler:]
Observing  -[UIApplication _openURL:originatingView:completionHandler:]
Observing  -[SUApplication application:openURL:sourceApplication:annotation:]
...
```

The list is very long and includes the methods we have already mentioned. If we trigger now one URL scheme, for example "igoat://" from Safari and accept to open it in the app we will see the following:

```javascript
[iPhone::iGoat-Swift]-> (0x1c4038280)  -[iGoat_Swift.AppDelegate application:openURL:options:]
application: <UIApplication: 0x101d0fad0>
openURL: igoat://
options: {
    UIApplicationOpenURLOptionsOpenInPlaceKey = 0;
    UIApplicationOpenURLOptionsSourceApplicationKey = "com.apple.mobilesafari";
}
0x18b5030d8 UIKit!__58-[UIApplication _applicationOpenURLAction:payload:origin:]_block_invoke
0x18b502a94 UIKit!-[UIApplication _applicationOpenURLAction:payload:origin:]
0x18b50c1fc UIKit!-[UIApplication _handleNonLaunchSpecificActions:forScene:withTransitionContext:completion:]
0x18b792f10 UIKit!-[__UICanvasLifecycleMonitor_Compatability activateEventsOnly:withContext:completion:]
0x18bf0e4b8 UIKit!__82-[_UIApplicationCanvas _transitionLifecycleStateWithTransitionContext:completion:]_block_invoke
0x18bf0e35c UIKit!-[_UIApplicationCanvas _transitionLifecycleStateWithTransitionContext:completion:]
0x18bc80294 UIKit!__125-[_UICanvasLifecycleSettingsDiffAction performActionsForCanvas:withUpdatedScene:settingsDiff:fromSettings:transitionContext:]_block_invoke
0x18be170ac UIKit!_performActionsWithDelayForTransitionContext
0x18bc80144 UIKit!-[_UICanvasLifecycleSettingsDiffAction performActionsForCanvas:withUpdatedScene:settingsDiff:fromSettings:transitionContext:]
0x18ba662cc UIKit!-[_UICanvas scene:didUpdateWithDiff:transitionContext:completion:]
0x18b908d5c UIKit!-[UIApplicationSceneClientAgent scene:handleEvent:withCompletion:]
0x18450a20c FrontBoardServices!__80-[FBSSceneImpl updater:didUpdateSettings:withDiff:transitionContext:completion:]_block_invoke.362
0x1817e1048 libdispatch.dylib!_dispatch_client_callout
0x1817e86c8 libdispatch.dylib!_dispatch_block_invoke_direct$VARIANT$mp
0x18453d9f4 FrontBoardServices!__FBSSERIALQUEUE_IS_CALLING_OUT_TO_A_BLOCK__
0x18453d698 FrontBoardServices!-[FBSSerialQueue _performNext]
RET: 0x1
```

Now we know that:

- The method `-[iGoat_Swift.AppDelegate application:openURL:options:]` gets called. As we have seen before, it is the recommended way and it is not deprecated.
- It gets our URL as a parameter: `igoat://`
- We also can verify the source application: `com.apple.mobilesafari`
- We can also know from where it was called, as expected from `-[UIApplication _applicationOpenURLAction:payload:origin:]`
- The method returns `0x1` which means `YES` ([the delegate successfully handled the request](https://developer.apple.com/documentation/uikit/uiapplicationdelegate/1623112-application?language=objc#return-value))

The call was successful and we see now that the iGoat app was open:

![iGoat Opened via URL Scheme](Images/Chapters/0x06h/iGoat_opened_via_url_scheme.jpg)

Notice that we can also see that the caller (source application) was Safari if we look in the upper-left corner of the screenshot.

###### Dynamically opening the link from the app itfelf (Frida REPL)

It is also interesting to see which other methods get called on the way. To change the result a little bit we will call the same URL Scheme from the iGoat app itself:

```javascript
$ frida -U iGoat-Swift --codeshare mrmacete/objc-method-observer

[iPhone::iGoat-Swift]-> function openURL(url) {
                            var UIApplication = ObjC.classes.UIApplication.sharedApplication();
                            var toOpen = ObjC.classes.NSURL.URLWithString_(url);
                            return UIApplication.openURL_(toOpen);
                        }

[iPhone::iGoat-Swift]-> observeSomething("*[* *openURL*]");
[iPhone::iGoat-Swift]-> openURL("iGoat://?contactNumber=123456789&message=hola")

(0x1c409e460)  -[__NSXPCInterfaceProxy__LSDOpenProtocol openURL:options:completionHandler:]
openURL: iGoat://?contactNumber=123456789&message=hola
options: nil
completionHandler: <__NSStackBlock__: 0x16fc89c38>
0x183befbec MobileCoreServices!-[LSApplicationWorkspace openURL:withOptions:error:]
0x10ba6400c
...
0x10a722f08
RET: nil

(0x1c422ee00)  -[LSApplicationWorkspace openURL:withOptions:error:]
openURL: iGoat://?contactNumber=123456789&message=hola
withOptions: nil
error: nil
0x10ba2400c
...
0x10a723084
RET: 0x1

(0x1c422ee00)  -[LSApplicationWorkspace openURL:withOptions:]
openURL: iGoat://?contactNumber=123456789&message=hola
withOptions: nil
0x18b501fd8 UIKit!-[UIApplication _openURL:]
0x10b80400c
...
0x10a723084
RET: 0x1

(0x1c422ee00)  -[LSApplicationWorkspace openURL:]
openURL: iGoat://?contactNumber=123456789&message=hola
0x18b501fd8 UIKit!-[UIApplication _openURL:]
0x10b80400c
...
0x10a723084
RET: 0x1

(0x101d0fad0)  -[UIApplication _openURL:]
_openURL: iGoat://?contactNumber=123456789&message=hola
0x10a610044
...
0x10a722e58
RET: 0x1

(0x101d0fad0)  -[UIApplication openURL:]
openURL: iGoat://?contactNumber=123456789&message=hola
0x10a610044
...
0x10a722e58
RET: 0x1

true
(0x1c4038280)  -[iGoat_Swift.AppDelegate application:openURL:options:]
application: <UIApplication: 0x101d0fad0>
openURL: iGoat://?contactNumber=123456789&message=hola
options: {
    UIApplicationOpenURLOptionsOpenInPlaceKey = 0;
    UIApplicationOpenURLOptionsSourceApplicationKey = "OWASP.iGoat-Swift";
}
0x18b5030d8 UIKit!__58-[UIApplication _applicationOpenURLAction:payload:origin:]_block_invoke
0x18b502a94 UIKit!-[UIApplication _applicationOpenURLAction:payload:origin:]
0x18b50c1fc UIKit!-[UIApplication _handleNonLaunchSpecificActions:forScene:withTransitionContext:completion:]
0x18bf0e480 UIKit!__82-[_UIApplicationCanvas _transitionLifecycleStateWithTransitionContext:completion:]_block_invoke
0x18bf0e35c UIKit!-[_UIApplicationCanvas _transitionLifecycleStateWithTransitionContext:completion:]
...
RET: 0x1
```

The output is truncated for better readability. This time you see that `UIApplicationOpenURLOptionsSourceApplicationKey` has changed to `OWASP.iGoat-Swift`, which makes sense. In addition, we see an overview of all `openURL`-like methods being called. This can be very useful for some scenarios as it will help you to decide what you next steps will be, e.g. which method you will hook or tamper with next.

###### Opening a link by navigating to a page and letting Safari open it (frida-trace)

We do it now with Safari and Telegram, but instead of giving it manually into the search bar, we will let Safari identify and process the URL scheme from a page containing one. Opening this link "https://telegram.me/fridadotre" will trigger this behaviour.

![Open this page in "Telegram"?](Images/Chapters/0x06h/open_in_telegram_via_urlscheme.png)

First of all we let frida-trace generate the stubs for us:

```bash
$ frida-trace -U Telegram -m "*[* *restorationHandler*]" -i "*open*Url*" -m "*[* *application*URL*]" -m "*[* openURL]"

...
7310 ms  -[UIApplication _applicationOpenURLAction: 0x1c44ff900 payload: 0x10c5ee4c0 origin: 0x0]
7311 ms     | -[AppDelegate application: 0x105a59980 openURL: 0x1c46ebb80 options: 0x1c0e222c0]
7312 ms     | $S10TelegramUI15openExternalUrl7account7context3url05forceD016presentationData
            18applicationContext20navigationController12dismissInputy0A4Core7AccountC_AA
            14OpenURLContextOSSSbAA012PresentationK0CAA0a11ApplicationM0C7Display010NavigationO0CSgyyctF()
```

Now we can simply modify by hand the stubs we are interested in, that is, the Objective-C method `application:openURL:options:`:

```javascript
// __handlers__/__AppDelegate_application_openUR_3679fadc.js

onEnter: function (log, args, state) {
    log("-[AppDelegate application: " + args[2] + " openURL: " + args[3] + " options: " + args[4] + "]");
    log("\tapplication :" + ObjC.Object(args[2]).toString());
    log("\topenURL :" + ObjC.Object(args[3]).toString());
    log("\toptions :" + ObjC.Object(args[4]).toString());
},
```

And the Swift method `$S10TelegramUI15openExternalUrl...`:

```javascript
// __handlers__/TelegramUI/_S10TelegramUI15openExternalUrl7_b1a3234e.js

  onEnter: function (log, args, state) {

    log("TelegramUI.openExternalUrl(account, url, presentationData," +
                "applicationContext, navigationController, dismissInput)");
    log("\taccount: " + ObjC.Object(args[1]).toString());
    log("\turl: " + ObjC.Object(args[2]).toString());
    log("\tpresentationData: " + args[3]);
    log("\tapplicationContext: " + ObjC.Object(args[4]).toString());
    log("\tnavigationController: " + ObjC.Object(args[5]).toString());
  },
```

The next time we run it, we see the following output:

```javascript
$ frida-trace -U Telegram -m "*[* *restorationHandler*]" -i "*open*Url*" -m "*[* *application*URL*]" -m "*[* openURL]"

  8144 ms  -[UIApplication _applicationOpenURLAction: 0x1c44ff900 payload: 0x10c5ee4c0 origin: 0x0]
  8145 ms     | -[AppDelegate application: 0x105a59980 openURL: 0x1c46ebb80 options: 0x1c0e222c0]
  8145 ms     | 	application: <Application: 0x105a59980>
  8145 ms     | 	openURL: tg://resolve?domain=fridadotre
  8145 ms     | 	options :{
                        UIApplicationOpenURLOptionsOpenInPlaceKey = 0;
                        UIApplicationOpenURLOptionsSourceApplicationKey = "com.apple.mobilesafari";
                    }
  8269 ms     |    | TelegramUI.openExternalUrl(account, url, presentationData, 
                                        applicationContext, navigationController, dismissInput)
  8269 ms     |    | 	account: nil
  8269 ms     |    | 	url: tg://resolve?domain=fridadotre
  8269 ms     |    | 	presentationData: 0x1c4c51741
  8269 ms     |    | 	applicationContext: nil
  8269 ms     |    | 	navigationController: TelegramUI.PresentationData
  8274 ms     | -[UIApplication applicationOpenURL:0x1c46ebb80]
```

There you can observe the following:

- it calls `application:openURL:options:` from the app delegate as expected
- the source application is Safari ("com.apple.mobilesafari")
- `application:openURL:options:` handles the URL but does not open it, it calls `TelegramUI.openExternalUrl` for that
- the URL being opened is `tg://resolve?domain=fridadotre`
- it uses the `tg://` custom URL scheme from Telegram


It is interesting to see that if you open the same link "https://telegram.me/fridadotre", click on cancel and the click on the link offered by the page itself "Open in the Telegram app, instead of opening via custom URL scheme it will open via Universal Links.

![Open in the Telegram app](Images/Chapters/0x06h/open_in_telegram_via_universallink.png)

You can try this and trace both methods like this:

```javascript
$ frida-trace -U Telegram -m "*[* *restorationHandler*]" -m "*[* *application*openURL*options*]"

// After clicking "Open" on the pop-up
           
 16374 ms  -[AppDelegate application :0x10556b3c0 openURL :0x1c4ae0080 options :0x1c7a28400]
 16374 ms  	application :<Application: 0x10556b3c0>
 16374 ms  	openURL :tg://resolve?domain=fridadotre
 16374 ms  	options :{
    UIApplicationOpenURLOptionsOpenInPlaceKey = 0;
    UIApplicationOpenURLOptionsSourceApplicationKey = "com.apple.mobilesafari";
}

// After clicking "Cancel" on the pop-up and "OPEN" in the page

406575 ms  -[AppDelegate application:0x10556b3c0 continueUserActivity:0x1c063d0c0 restorationHandler:0x16f27a898]
406575 ms  	application:<Application: 0x10556b3c0>
406575 ms  	continueUserActivity:<NSUserActivity: 0x1c063d0c0>
406575 ms  		webpageURL:https://telegram.me/fridadotre
406575 ms  		activityType:NSUserActivityTypeBrowsingWeb
406575 ms  		userInfo:{
}
406575 ms  	restorationHandler:<__NSStackBlock__: 0x16f27a898>
```

###### Testing for Deprecated Methods

Search for deprecated methods like:

- [`application:handleOpenURL:`](https://developer.apple.com/documentation/uikit/uiapplicationdelegate/1622964-application?language=objc)
- [`openURL:`](https://developer.apple.com/documentation/uikit/uiapplication/1622961-openurl?language=objc)
- [`application:openURL:sourceApplication:annotation:`](https://developer.apple.com/documentation/uikit/uiapplicationdelegate/1623073-application)

Simply use frida-trace to see if any of those methods is being used.

##### Testing URL Schemes Source Validation

A way to discard or confirm validation could be by hooking typical methods that might be used for that. For example [`isEqualToString:`](https://developer.apple.com/documentation/foundation/nsstring/1407803-isequaltostring):

```javascript
// - (BOOL)isEqualToString:(NSString *)aString;

var isEqualToString = ObjC.classes.NSString["- isEqualToString:"];

Interceptor.attach(isEqualToString.implementation, {
  onEnter: function(args) {
    var message = ObjC.Object(args[2]);
    console.log(message)
  }
});
```

If we apply this hook and call the URL scheme again:

```javascript
$ frida -U iGoat-Swift

[iPhone::iGoat-Swift]-> var isEqualToString = ObjC.classes.NSString["- isEqualToString:"];

                    Interceptor.attach(isEqualToString.implementation, {
                      onEnter: function(args) {
                        var message = ObjC.Object(args[2]);
                        console.log(message)
                      }
                    });
{}
[iPhone::iGoat-Swift]-> openURL("iGoat://?contactNumber=123456789&message=hola")
true
nil
```

Nothing happens. This tells us already that this method is not being used for that as we cannot find any *app-package-looking* string like `OWASP.iGoat-Swift` or `com.apple.mobilesafari` between the hook and the text of the tweet. However, consider that we are just probing one method, the app might be using other approach for the comparison.


##### Fuzzing URL Schemes

If the app parses parts of the URL, you can also perform input fuzzing to detect memory corruption bugs.

What we have learned above can be now used to build your own fuzzer on the language of your choice, e.g. in Python and call the `openURL` using [Frida's RPC](https://www.frida.re/docs/javascript-api/#rpc). That fuzzer should do the following:

- generate payloads
- for each of them call `openURL`
- and check if the app generates a crash report (`.ips`) in `/private/var/mobile/Library/Logs/CrashReporter`

The [FuzzDB](https://github.com/fuzzdb-project/fuzzdb) project offers fuzzing dictionaries that you can use as payloads.

###### Using Frida

Doing this with Frida is pretty easy, you can refer to this [blog post](https://grepharder.github.io/blog/0x03_learning_about_universal_links_and_fuzzing_url_schemes_on_ios_with_frida.html) to see an example that fuzzes the iGoat-Swift app (working on iOS 11.1.2).

Before running the fuzzer we need the URL schemes as inputs. From the static analysis we know that the iGoat-Swift app supports the following URL scheme and parameters: `iGoat://?contactNumber={0}&message={0}`.


```javascript
$ frida -U SpringBoard -l ios-url-scheme-fuzzing.js
[iPhone::SpringBoard]-> fuzz("iGoat", "iGoat://?contactNumber={0}&message={0}")
Watching for crashes from iGoat...
No logs were moved.
Opened URL: iGoat://?contactNumber=0&message=0
OK!
Opened URL: iGoat://?contactNumber=1&message=1
OK!
Opened URL: iGoat://?contactNumber=-1&message=-1
OK!
Opened URL: iGoat://?contactNumber=null&message=null
OK!
Opened URL: iGoat://?contactNumber=nil&message=nil
OK!
Opened URL: iGoat://?contactNumber=99999999999999999999999999999999999
&message=99999999999999999999999999999999999
OK!
Opened URL: iGoat://?contactNumber=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
...
&message=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
...
OK!
Opened URL: iGoat://?contactNumber=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
...
&message=AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA
...
OK!
Opened URL: iGoat://?contactNumber='&message='
OK!
Opened URL: iGoat://?contactNumber=%20d&message=%20d
OK!
Opened URL: iGoat://?contactNumber=%20n&message=%20n
OK!
Opened URL: iGoat://?contactNumber=%20x&message=%20x
OK!
Opened URL: iGoat://?contactNumber=%20s&message=%20s
OK!
```

The script will detect if a crash occurred. On this run it did not detect any crashed but for other apps this could be the case. We would be able to inspect the crash reports in `/private/var/mobile/Library/Logs/CrashReporter` or in `/tmp` if it was moved by the script.

###### Using IDB

In the "URL Handlers" section, go to the "Fuzzer" tab. On the left side default IDB payloads are listed. Once you have generated your payload list (e.g. using FuzzDB), go to the "Fuzz Template" section in the left bottom panel and define a template. Use `$@$` to define an injection point, for example:

```
myURLscheme://$@$
```

While the URL scheme is being fuzzed, watch the logs (in Xcode, go to Window -> Devices -> *click on your device* -> *bottom console contains logs*) to observe the impact of each payload. The history of used payloads is on the right side of the IDB "Fuzzer" tab.


### Testing iOS WebViews

#### Overview

WebViews are in-app browser components for displaying interactive web content. They can be used to embed web content directly into an app's user interface. iOS WebViews support JavaScript execution by default, so script injection and Cross-Site Scripting attacks can affect them.

##### `UIWebView`

[`UIWebView`](https://developer.apple.com/reference/uikit/uiwebview "UIWebView reference documentation") (for iOS versions 7.1.2 and older) is deprecated and should not be used. Make sure that either `WKWebView` or `SFSafariViewController` are used to embed web content. In addition to that, JavaScript cannot be disabled for `UIWebView` which is another reason to refrain from using it.

##### `WKWebView`

[`WKWebView`](https://developer.apple.com/reference/webkit/wkwebview "WKWebView reference documentation") (for iOS in version 8.0 and later) is the appropriate choice for extending app functionality, controlling displayed content (i.e., prevent the user from navigating to arbitrary URLs) and customizing.

`WKWebView` comes with several security advantages over `UIWebView`:

- JavaScript is enabled by default but thanks to the `JavaScriptEnabled` property of `WKWebView`, it can be completely disabled, preventing all script injection flaws.
- The `JavaScriptCanOpenWindowsAutomatically` can be used to prevent JavaScript from opening new windows, such as pop-ups.
- the `hasOnlySecureContent` property can be used to verify resources loaded by the WebView are retrieved through encrypted connections.
- `WKWebView` implements out-of-process rendering, so memory corruption bugs won't affect the main app process.

`WKWebView` also increases the performance of apps that are using WebViews significantly, through the Nitro JavaScript engine [#THIEL].

###### `WKWebView`'s JavaScript Bridge

JavaScript code can still send messages back to the native app but in contrast to `UIWebView`, it is not possible to directly reference the `JSContext` of a `WKWebView`. Instead, communication is implemented using a messaging system and using the `postMessage` function.

`postMessage` automatically serializes JavaScript objects into native Objective-C or Swift objects. Message handlers are configured using the method [`add(_ scriptMessageHandler:name:)`](https://developer.apple.com/documentation/webkit/wkusercontentcontroller/1537172-add).

See Section "Determining Whether Native Methods Are Exposed Through WebViews" below for more information.


##### `SFSafariViewController`

[`SFSafariViewController`](https://developer.apple.com/documentation/safariservices/sfsafariviewcontroller) should be used to provide a generalized web viewing experience. However, JavaScript cannot be disabled in `SFSafariViewController` and this is one of the reason why you should recommend usage of `WKWebView` when the goal is extending the app's user interface.

> Note that `SFSafariViewController` also shares cookies and other website data with Safari.


#### Static Analysis

Look out for usages of the above mentioned WebView classes.

##### Testing JavaScript Configuration

As a best practice, JavaScript should be disabled in a `WKWebView` unless it is explicitly required. To verify that JavaScript was properly disabled search the project for usages of `WKPreferences` and ensure that the `javaScriptEnabled` property is set to `false`:

```swift
let webPreferences = WKPreferences()
webPreferences.javaScriptEnabled = false
```
> Dynamic Analysis:
In order to test this you could use Frida and verify that the `javaScriptEnabled` property is set to `false` for all instances.

##### Testing for Local File Inclusion

WebViews can load content remotely and locally from the app data directory. If the content is loaded locally, users should not be able to change the filename or path from which the file is loaded, and they shouldn't be able to edit the loaded file.

Check the source code for WebViews usage. If you can identify a WebView instance, check whether any local files have been loaded (`example_file.html` in the below example).

```objc

- (void)viewDidLoad
{
    [super viewDidLoad];
    WKWebViewConfiguration *configuration = [[WKWebViewConfiguration alloc] init];

    self.webView = [[WKWebView alloc] initWithFrame:CGRectMake(10, 20, CGRectGetWidth([UIScreen mainScreen].bounds) - 20, CGRectGetHeight([UIScreen mainScreen].bounds) - 84) configuration:configuration];
    self.webView.navigationDelegate = self;
    [self.view addSubview:self.webView];

    NSString *filePath = [[NSBundle mainBundle] pathForResource:@"example_file" ofType:@"html"];
    NSString *html = [NSString stringWithContentsOfFile:filePath encoding:NSUTF8StringEncoding error:nil];
    [self.webView loadHTMLString:html baseURL:[NSBundle mainBundle].resourceURL];
}

```

Check the `baseURL` for dynamic parameters that can be manipulated (leading to local file inclusion).

##### Testing for HTTPS-Only Content

In `WKWebView`s it is possible to detect mixed content or content that was completely loaded via HTTP. By using the method `hasOnlySecureContent` it can be ensured that only content via HTTPS is shown, otherwise an alert is displayed to the user, see page 159 and 160 in [#THIEL] for an example.  

#### Dynamic Analysis

To simulate an attack, inject your own JavaScript into the WebView with an interception proxy. Attempt to access local storage and any native methods and properties that might be exposed to the JavaScript context.

In a real-world scenario, JavaScript can only be injected through a permanent backend Cross-Site Scripting vulnerability or a MITM attack. See the OWASP [XSS cheat sheet](https://goo.gl/x1mMMj "XSS (Cross-Site Scripting) Prevention Cheat Sheet") and the chapter "Testing Network Communication" for more information.


### Testing WebView Protocol Handlers

#### Overview

Several default schemes are available that are being interpreted in a WebView on iOS, for example:

- http(s)://
- file://
- tel://

WebViews can load remote content from an endpoint, but they can also load local content from the app data directory. If the local content is loaded, the user shouldn't be able to influence the filename or the path used to load the file, and users shouldn't be able to edit the loaded file.

#### Static Analysis

Check the source code for WebView usage.

On Android, there is an option to disable file scheme URLs (`setAllowFileAccess`). Unfortunately, on iOS it is enabled by default and **cannot be disabled**.

The following WebView settings control resource access:

- `allowFileAccessFromFileURLs`
- `allowUniversalAccessFromFileURLs`
- `allowingReadAccessToURL`

`WKWebView` default `allowFileAccessFromFileURLs` and `allowUniversalAccessFromFileURLs` option is `false`.

However, it is possible to set the [undocumented](https://github.com/WebKit/webkit/blob/master/Source/WebKit/UIProcess/API/Cocoa/WKPreferences.mm#L470) `allowFileAccessFromFileURLs` property in a WebView. Here is an example:

Objective-C:

```objc

[webView.configuration.preferences setValue:@YES forKey:@"allowFileAccessFromFileURLs"];

```

Swift:

```swift

webView.configuration.preferences.setValue(true, forKey: "allowFileAccessFromFileURLs")

```

By default `WKWebView` disables file access. If one or more of the above methods are activated, you should determine whether they are really necessary for the app to work properly.

Please also verify which WebView class is used. Remember that `WKWebView` should be used nowadays, as `UIWebView` is deprecated.

If a WebView instance can be identified, find out whether local files are loaded with the [`loadFileURL(_:allowingReadAccessTo:)`](https://developer.apple.com/documentation/webkit/wkwebview/1414973-loadfileurl "loadFileURL") method.

Objective-C:

```objc

[self.wk_webview loadFileURL:url allowingReadAccessToURL:readAccessToURL];

```

Swift:

```swift

webview.loadFileURL(url, allowingReadAccessTo: bundle.resourceURL!)

```

The URL specified in `loadFileURL` should be checked for dynamic parameters that can be manipulated as that may lead to [local file inclusion](https://en.wikipedia.org/wiki/File_inclusion_vulnerability#Local_file_inclusion).

##### telephone number detection

In Safari on iOS, telephone number detection is on by default. However, you might want to turn it off if your HTML page contains numbers that can be interpreted as phone numbers, but are not phone numbers, or to prevent the DOM document from being modified when parsed by the browser. To turn off telephone number detection in Safari on iOS, use the format-detection meta tag (`<meta name = "format-detection" content = "telephone=no">`). An example of this can be found [here](https://developer.apple.com/library/archive/featuredarticles/iPhoneURLScheme_Reference/PhoneLinks/PhoneLinks.html#//apple_ref/doc/uid/TP40007899-CH6-SW2). Phone links should be then used (e.g. `<a href="tel:1-408-555-5555">1-408-555-5555</a>`) to explicitly create a link.

Use the following best practices as defensive-in-depth measures:

- create a whitelist that defines local and remote web pages and URL schemes that are allowed to be loaded.
- create checksums of the local HTML/JavaScript files and check them while the app is starting up. [Minify JavaScript files](https://en.wikipedia.org/wiki/Minification_(programming)) to make them harder to read.

#### Dynamic Analysis

To identify the usage of protocol handlers, look for ways to access files from the file system and trigger phone calls while you're using the app.

If it's possible to load local files via a WebView, the app might be vulnerable to directory traversal attacks. This would allow access to all files within the sandbox or even to escape the sandbox with full access to the file system (if the device is jailbroken).  

It should therefore be verified if a user can change the filename or path from which the file is loaded, and they shouldn't be able to edit the loaded file.


### Determining Whether Native Methods Are Exposed Through WebViews

#### Overview

Starting from iOS version 7.0, Apple introduced APIs that allow communication between the JavaScript runtime in the WebView and the native Swift or Objective-C objects. If these APIs are used carelessly, important functionality might be exposed to attackers who manage to inject malicious scripts into the WebView (e.g., through a successful Cross-Site Scripting attack).

#### Static Analysis

Both `UIWebView` and `WKWebView` provide a means of communication between the WebView and the native app. Any important data or native functionality exposed to the WebView JavaScript engine would also be accessible to rogue JavaScript running in the WebView.

Since iOS 7, the JavaScriptCore framework provides an Objective-C wrapper to the WebKit JavaScript engine. This makes it possible to execute JavaScript from Swift and Objective-C, as well as making Objective-C and Swift objects accessible from the JavaScript runtime. A JavaScript execution environment is represented by a `JSContext` object. Look out for code that maps native objects to the `JSContext` associated with a WebView and analyze what functionality it exposes, for example no sensitive data should be accessible and exposed to WebViews.


##### Testing `UIWebView` JavaScript to Native Bridges

In Objective-C, the `JSContext` associated with a `UIWebView` is obtained as follows:

```objc

[webView valueForKeyPath:@"documentView.webView.mainFrame.javaScriptContext"]

```

There are two fundamental ways of how native code and JavaScript can communicate:

- **JSContext**: When an Objective-C or Swift block is assigned to an identifier in a `JSContext`, JavaScriptCore automatically wraps the block in a JavaScript function.
- **JSExport protocol**: Properties, instance methods and class methods declared in a `JSExport`-inherited protocol are mapped to JavaScript objects that are available to all JavaScript code. Modifications of objects that are in the JavaScript environment are reflected in the native environment.

Note that only class members defined in the `JSExport` protocol are made accessible to JavaScript code.

##### Testing `WKWebView` JavaScript to Native Bridges

Verify if a JavaScript to Native bridge exists by searching for `WKScriptMessageHandler` and check all exposed methods. Then verify how the methods are called. The following example from [WheresMyBrowser iOS App](https://github.com/authenticationfailure/WheresMyBrowser.iOS/blob/b8d4abda4000aa509c7a5de79e5c90360d1d0849/WheresMyBrowser/WKWebViewPreferencesManager.swift#L98) demonstrates this.

First we see how the JavaScript bridge is enabled:

```swift
func enableJavaScriptBridge(_ enabled: Bool) {
        options_dict["javaScriptBridge"]?.value = enabled
        let userContentController = wkWebViewConfiguration.userContentController
        userContentController.removeScriptMessageHandler(forName: "javaScriptBridge")

        if enabled {
                let javaScriptBridgeMessageHandler = JavaScriptBridgeMessageHandler()
                userContentController.add(javaScriptBridgeMessageHandler, name: "javaScriptBridge")
        }
}
```

Adding a script message handler with name `"name"` (or `"javaScriptBridge"` in the  example above) causes the JavaScript function `window.webkit.messageHandlers.myJavaScriptMessageHandler.postMessage()` to be defined in all frames in all web views that use the user content controller. It can be then [used form the HTML file like this](https://github.com/authenticationfailure/WheresMyBrowser.iOS/blob/d4e2d9efbde8841bf7e4a8800418dda6bb116ec6/WheresMyBrowser/web/WKWebView/scenario3.html#L33).

```javascript
function invokeNativeOperation() {
        value1 = document.getElementById("value1").value
        value2 = document.getElementById("value2").value
        window.webkit.messageHandlers.javaScriptBridge.postMessage(["multiplyNumbers", value1, value2]);
}
```

The called function resides in [`JavaScriptBridgeMessageHandler.swift`](https://github.com/authenticationfailure/WheresMyBrowser.iOS/blob/b8d4abda4000aa509c7a5de79e5c90360d1d0849/WheresMyBrowser/JavaScriptBridgeMessageHandler.swift#L29):

```swift
class JavaScriptBridgeMessageHandler: NSObject, WKScriptMessageHandler {

...

case "multiplyNumbers":

        let arg1 = Double(messageArray[1])!
        let arg2 = Double(messageArray[2])!
        result = String(arg1 * arg2)
...

let javaScriptCallBack = "javascriptBridgeCallBack('\(functionFromJS)','\(result)')"
message.webView?.evaluateJavaScript(javaScriptCallBack, completionHandler: nil)
```

The problem here is that the `JavaScriptBridgeMessageHandler` not only contains that function, it exposes a sensitive function:

```swift
case "getSecret":
        result = "XSRSOGKC342"
```

#### Dynamic Analysis

Dynamic analysis of the app can show you which HTML or JavaScript files are loaded while using the app. You would need to find all WebView in the iOS app in order to get an overview of the potential attack surface.

Usage of the `JSContext` / `JSExport` for `UIWebView` and `WKScriptMessageHandler` for `WKWebView` should be ideally identified through reverse engineering and static analysis, as well as which functions are exposed and present in a WebView. The procedure for exploiting the functions starts with producing a JavaScript payload and injecting it into the file that the app is requesting. The injection can be accomplished via a MITM attack.

In the previous example, it was trivial to get the secret value by performing reverse engineering but imagine that the exposed function retrieves the secret from secure storage. As some content is loaded insecurely from the Internet over HTTP we could apply the previous technique and get the secret by injecting the following payload:

```javascript
var res = window.webkit.messageHandlers.javaScriptBridge.postMessage(["getSecret"]);
document.getElementById("result").innerHTML=res;
```

See another example for a vulnerable iOS app and function that is exposed to a WebView in [#THIEL] page 156.


### Testing Object Persistence

#### Overview

There are several ways to persist an object on iOS:

##### Object Encoding

iOS comes with two protocols for object encoding and decoding for Objective-C or `NSObject`s: `NSCoding` and `NSSecureCoding`. When a class conforms to either of the protocols, the data is serialized to `NSData`: a wrapper for byte buffers. Note that `Data` in Swift is the same as `NSData` or its mutable counterpart: `NSMutableData`. The `NSCoding` protocol declares the two methods that must be implemented in order to encode/decode its instance-variables. A class using `NSCoding` needs to implement `NSObject` or be annotated as an @objc class. The `NSCoding` protocol requires to implement encode and init as shown below.

```swift
class CustomPoint: NSObject, NSCoding {

    //required by NSCoding:
    func encode(with aCoder: NSCoder) {
        aCoder.encode(x, forKey: "x")
        aCoder.encode(name, forKey: "name")
    }

    var x: Double = 0.0
    var name: String = ""

    init(x: Double, name: String) {
            self.x = x
            self.name = name
    }

    // required by NSCoding: initialize members using a decoder.
    required convenience init?(coder aDecoder: NSCoder) {
            guard let name = aDecoder.decodeObject(forKey: "name") as? String
                    else {return nil}
            self.init(x:aDecoder.decodeDouble(forKey:"x"),
                                name:name)
    }

    //getters/setters/etc.
}
```

The issue with `NSCoding` is that the object is often already constructed and inserted before you can evaluate the class-type. This allows an attacker to easily inject all sorts of data. Therefore, the `NSSecureCoding` protocol has been introduced. When conforming to [`NSSecureCoding`](https://developer.apple.com/documentation/foundation/NSSecureCoding) you need to include:

```swift

static var supportsSecureCoding: Bool {
        return true
}
```

when `init(coder:)` is part of the class. Next, when decoding the object, a check should be made, e.g.:

```swift
let obj = decoder.decodeObject(of:MyClass.self, forKey: "myKey")
```

The conformance to `NSSecureCoding` ensures that objects being instantiated are indeed the ones that were expected. However, there are no additional integrity checks done over the data and the data is not encrypted. Therefore, any secret data needs additional encryption and data of which the integrity must be protected, should get an additional HMAC.

Note, when `NSData` (Objective-C) or the keyword `let` (Swift) is used: then the data is immutable in memory and cannot be easily removed.


##### Object Archiving with NSKeyedArchiver

`NSKeyedArchiver` is a concrete subclass of `NSCoder` and provides a way to encode objects and store them in a file. The `NSKeyedUnarchiver` decodes the data and recreates the original data. Let's take the example of the `NSCoding` section and now archive and unarchive them:

```swift

// archiving:
NSKeyedArchiver.archiveRootObject(customPoint, toFile: "/path/to/archive")

// unarchiving:
guard let customPoint = NSKeyedUnarchiver.unarchiveObjectWithFile("/path/to/archive") as? CustomPoint else { return nil }

```

When decoding a keyed archive, because values are requested by name, values can be decoded out of sequence or not at all. Keyed archives, therefore, provide better support for forward and backward compatibility. This means that an archive on disk could actually contain additional data which is not detected by the program, unless the key for that given data is provided at a later stage.

Note that additional protection needs to be in place to secure the file in case of confidential data, as the data is not encrypted within the file. See the "Data Storage on iOS" chapter for more details.

##### Codable

With Swift 4, the `Codable` type alias arrived: it is a combination of the `Decodable` and `Encodable` protocols. A `String`, `Int`, `Double`, `Date`, `Data` and `URL` are `Codable` by nature: meaning they can easily be encoded and decoded without any additional work. Let's take the following example:

```swift
struct CustomPointStruct:Codable {
    var x: Double
    var name: String
}
```

By adding `Codable` to the inheritance list for the `CustomPointStruct` in the example, the methods `init(from:)` and `encode(to:)` are automatically supported. Fore more details about the workings of `Codable` check [the Apple Developer Documentation](https://developer.apple.com/documentation/foundation/archives_and_serialization/encoding_and_decoding_custom_types "Codable").
The `Codable`s can easily be encoded / decoded into various representations: `NSData` using `NSCoding`/`NSSecureCoding`, JSON, Property Lists, XML, etc. See the subsections below for more details.

##### JSON and Codable
There are various ways to encode and decode JSON within iOS by using different 3rd party librariesL
- [Mantle](https://github.com/Mantle/Mantle "Mantle"),
- [JSONModel library](https://github.com/jsonmodel/jsonmodel "JSONModel"),
- [SwiftyJSON library](https://github.com/SwiftyJSON/SwiftyJSON "SwiftyJSON"),
- [ObjectMapper library](https://github.com/Hearst-DD/ObjectMapper, "ObjectMapper library"),
- [JSONKit](https://github.com/johnezang/JSONKit "JSONKit"),
- [JSONModel](https://github.com/JSONModel/JSONModel "JSONModel"),
- [YYModel](https://github.com/ibireme/YYModel "YYModel"),
- [SBJson 5](https://github.com/ibireme/YYModel "SBJson 5"),
- [Unbox](https://github.com/JohnSundell/Unbox "Unbox"),
- [Gloss](https://github.com/hkellaway/Gloss "Gloss"),
- [Mapper](https://github.com/lyft/mapper "Mapper"),
- [JASON](https://github.com/delba/JASON "JASON"),
- [Arrow](https://github.com/freshOS/Arrow "Arrow").

The libraries differ in their support for certain versions of Swift and Objective-C, whether they return (im)mutable results, speed, memory consumption and actual library size.
Again, note in case of immutability: confidential information cannot be removed from memory easily.

Next, Apple provides support for JSON encoding/decoding directly by combining `Codable` together with a `JSONEncoder` and a `JSONDecoder`:

```swift
struct CustomPointStruct:Codable {
    var x: Double
    var name: String
}

let encoder = JSONEncoder()
encoder.outputFormatting = .prettyPrinted

let test = CustomPointStruct(x: 10, name: "test")
let data = try encoder.encode(test)
print(String(data: data, encoding: .utf8)!)
// Prints:
// {
//   "x" : 10,
//   "name" : "test"
// }

```

JSON itself can be stored anywhere, e.g., a (NoSQL) database or a file. You just need to make sure that any JSON that contains secrets has been appropriately protected (e.g., encrypted/HMACed). See the "Data Storage on iOS" chapter for more details.


##### Property Lists and Codable

You can persist objects to *property lists* (also called plists in previous sections). You can find two examples below of how to use it:

```swift

// archiving:
let data = NSKeyedArchiver.archivedDataWithRootObject(customPoint)
NSUserDefaults.standardUserDefaults().setObject(data, forKey: "customPoint")

// unarchiving:

if let data = NSUserDefaults.standardUserDefaults().objectForKey("customPoint") as? NSData {
    let customPoint = NSKeyedUnarchiver.unarchiveObjectWithData(data)
}

```

In this first example, the `NSUserDefaults` are used, which is the primary *property list*. We can do the same with the `Codable` version:

```swift

struct CustomPointStruct:Codable {
    var x: Double
    var name: String
}

var points: [CustomPointStruct] = [
    CustomPointStruct(x: 1, name "test"),
    CustomPointStruct(x: 2, name "test"),
    CustomPointStruct(x: 3, name "test"),
]

UserDefaults.standard.set(try? PropertyListEncoder().encode(points), forKey:"points")
if let data = UserDefaults.standard.value(forKey:"points") as? Data {
    let points2 = try? PropertyListDecoder().decode(Array<CustomPointStruct>.self, from: data)
}

```

Note that **`plist` files are not meant to store secret information**. They are designed to hold user preferences for an app.

##### XML

There are multiple ways to do XML encoding. Similar to JSON parsing, there are various third party libraries, such as:

- [Fuzi](https://github.com/cezheng/Fuzi "Fuzi")
- [Ono](https://github.com/mattt/Ono "Ono")
- [AEXML](https://github.com/tadija/AEXML "AEXML")
- [RaptureXML](https://github.com/ZaBlanc/RaptureXML "RaptureXML")
- [SwiftyXMLParser](https://github.com/yahoojapan/SwiftyXMLParser "SwiftyXMLParser")
- [SWXMLHash](https://github.com/drmohundro/SWXMLHash "SWXMLHash")

They vary in terms of speed, memory usage, object persistence and more important: differ in how they handle XML external entities. See [XXE in the Apple iOS Office viewer](https://nvd.nist.gov/vuln/detail/CVE-2015-3784 "CVE-2015-3784") as an example. Therefore, it is key to disable external entity parsing if possible. See the [OWASP XXE prevention cheatsheet](https://goo.gl/86epVd "XXE prevention cheatsheet") for more details.
Next to the libraries, you can make use of Apple's [XMLParser class](https://developer.apple.com/documentation/foundation/xmlparser "XMLParser")

There are various ORM-like solutions for iOS. The first one is [Realm](https://realm.io/docs/swift/latest/ "Realm"), which comes with its own storage engine. Realm has settings to encrypt the data as explained in [Realm's documentation](https://academy.realm.io/posts/tim-oliver-realm-cocoa-tutorial-on-encryption-with-realm/ "Enable encryption"). This allows for handling secure data. Note that the encryption is turned off by default.

##### ORM (Coredata and Realm)
There are various ORM-like solutions for iOS. The first one is [Realm](https://realm.io/docs/swift/latest/ "Realm"), which comes with its own storage engine. Realm has settings to encrypt the data as explained in [Realm's documentation](https://academy.realm.io/posts/tim-oliver-realm-cocoa-tutorial-on-encryption-with-realm/ "Enable encryption"). This allows for handling secure data. Note that the encryption is turned off by default.

##### Protocol Buffers

[Protocol Buffers](https://developers.google.com/protocol-buffers/ "Google Documentation") by Google, are a platform- and language-neutral mechanism for serializing structured data by means of the [Binary Data Format](https://developers.google.com/protocol-buffers/docs/encoding "Encoding"). They are available for iOS by means of the [Protobuf](https://github.com/apple/swift-protobuf "Protobuf") library.
There have been a few vulnerabilities with Protocol Buffers, such as [CVE-2015-5237](https://www.cvedetails.com/cve/CVE-2015-5237/ "CVE-2015-5237").
Note that **Protocol Buffers do not provide any protection for confidentiality** as no built-in encryption is available.


#### Static Analysis

All different flavors of object persistence share the following concerns:

- If you use object persistence to store sensitive information on the device, then make sure that the data is encrypted: either at the database level, or specifically at the value level.
- Need to guarantee the integrity of the information? Use an HMAC mechanism or sign the information stored. Always verify the HMAC/signature before processing the actual information stored in the objects.
- Make sure that keys used in the two notions above are safely stored in the KeyChain and well protected. See the Data Storage section for more details.
- Ensure that the data within the deserialized object is carefully validated before it is actively used (e.g., no exploit of business/application logic).
- Do not use persistence mechanisms that use [Runtime Reference](https://developer.apple.com/library/archive/#documentation/Cocoa/Reference/ObjCRuntimeRef/Reference/reference.html "Objective-C runtime reference") to serialize/deserialize objects in high risk applications, as the attacker might be able to manipulate the steps to execute business logic via this mechanism (See anti-reverse-engineering chapter for more details).
- Note that in Swift 2 and beyond, the [Mirror](https://developer.apple.com/documentation/swift/mirror "Mirror") can be used to read parts of an object, but cannot be used to write against the object.

#### Dynamic Analysis

There are several ways to perform dynamic analysis:

- For the actual persistence: Use the techniques described in the "Data Storage on iOS" chapter.
- For the serialization itself: use a debug build or use Frida / objection to see how the serialization methods are handled (e.g., whether the application crashes or extra information can be extracted by enriching the objects).


### References

- [#THIEL] Thiel, David. iOS Application Security: The Definitive Guide for Hackers and Developers (Kindle Locations 3394-3399). No Starch Press. Kindle Edition.
- Security Flaw with UIWebView - (https://medium.com/ios-os-x-development/security-flaw-with-uiwebview-95bbd8508e3c "Security Flaw with UIWebView")
- Learning about Universal Links and Fuzzing URL Schemes on iOS with Frida - (https://grepharder.github.io/blog/0x03_learning_about_universal_links_and_fuzzing_url_schemes_on_ios_with_frida.html)

#### OWASP Mobile Top 10 2016

- M1 - Improper Platform Usage - https://www.owasp.org/index.php/Mobile_Top_10_2016-M1-Improper_Platform_Usage
- M7 - Poor Code Quality - https://www.owasp.org/index.php/Mobile_Top_10_2016-M7-Poor_Code_Quality

#### OWASP MASVS

- V6.1: "The app only requests the minimum set of permissions necessary."
- V6.3: "The app does not export sensitive functionality via custom URL schemes, unless these mechanisms are properly protected."
- V6.5: "JavaScript is disabled in WebViews unless explicitly required."
- V6.6: "WebViews are configured to allow only the minimum set of protocol handlers required (ideally, only https is supported). Potentially dangerous handlers, such as file, tel and app-id, are disabled."
- V6.7: "If native methods of the app are exposed to a WebView, verify that the WebView only renders JavaScript contained within the app package."
- V6.8: "Object serialization, if any, is implemented using safe serialization APIs."

#### CWE

- CWE-79 - Improper Neutralization of Input During Web Page Generation https://cwe.mitre.org/data/definitions/79.html
- CWE-200 - Information Leak / Disclosure
- CWE-939 - Improper Authorization in Handler for Custom URL Scheme

#### Tools

- Frida - https://www.frida.re/
- frida-trace - https://www.frida.re/docs/frida-trace/
- objection - https://github.com/sensepost/objection
- ObjC Method oObserver - https://codeshare.frida.re/@mrmacete/objc-method-observer/
- IDB - https://www.idbtool.com/
- Needle  https://github.com/mwrlabs/needle

#### Regarding Object Persistence in iOS

- https://developer.apple.com/documentation/foundation/NSSecureCoding
- https://developer.apple.com/documentation/foundation/archives_and_serialization?language=swift
- https://developer.apple.com/documentation/foundation/nskeyedarchiver
- https://developer.apple.com/documentation/foundation/nscoding?language=swift,https://developer.apple.com/documentation/foundation/NSSecureCoding?language=swift
- https://developer.apple.com/documentation/foundation/archives_and_serialization/encoding_and_decoding_custom_types
- https://developer.apple.com/documentation/foundation/archives_and_serialization/using_json_with_custom_types
- https://developer.apple.com/documentation/foundation/jsonencoder
- https://medium.com/if-let-swift-programming/migrating-to-codable-from-nscoding-ddc2585f28a4
- https://developer.apple.com/documentation/foundation/xmlparser
