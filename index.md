# Upgrade React Native Library

* Follow [Upgrade based on Git](https://facebook.github.io/react-native/docs/upgrading.html#upgrade-based-on-git)
* In the last step of the doc above, resolve the merge conflicts based on two rules
  * Keep "our" 3rd party library
  * Use "their" latest react native settings

# Usage of React Native Components

Tips and tricks of using some react native components/library

## react-navigation

* The message we see in console every time we navigate somewhere comes from [here](https://reactnavigation.org/docs/navigators/#onNavigationStateChange-prevState-newState-action)
* Set headerMode to 'none' for more convenient customization and powerful control of nav bar header
* Customize tab bar by hard-coding in src/views/TabView
* There is no intuitive way of doing popToRoute as the deprecated Navigator.  The solution is pass down the right screen key and use goBack(key).  See [this, note what mu1ex said](https://github.com/react-community/react-navigation/issues/285).  Nonetheless, there exists a flashing screen issue.  Check [this](https://github.com/react-community/react-navigation/pull/905) for potential solution.

## LayoutAnimation

* Very good API for doing animation, such as the zp-up/zip-down effect in CheckUp App
* See [this Jianshu article](http://www.jianshu.com/p/3ce1d27fc246)

## react-native-sortable-listview

* Not working if using react-navigation Modal mode for the component where this library is used.  Check [this github issue](https://github.com/deanmcpherson/react-native-sortable-listview/issues/75)

## react-native-video-player

* if playing, go back will trigger setState on component that is already unmounted, how to resolve? Use media-kit

## react-native-media-kit

* the right way to get the ref. Caveat: if you do this for the same COMPONENT more than once somwhere, the old ref will be null and only the new ref is not null. More info can be found [here](https://facebook.github.io/react/docs/refs-and-the-dom.html)

```javascript
ref={vp => {this._vps[rowId] = vp;}}
```

## react-native-image-crop-picker

* need to handle taking a video/photo and shouldSaveToRoll
* quality and time-limit of taking video, need handle compressVideo
* need network YES to download photos from iCloud
* If I do not releasePlayer in willMoveToWindow and willMoveToSuperview, I will get AVPlayerItemStatusFailed
* image resolution (PHAsset request option synchronous = false)
* QBImagePickerController to support Moments: need revision in AlbumViewController.m
* When archiving with a build other than release, be aware of the link error that the frameworks cannot be found, see [this github issue](https://github.com/ivpusic/react-native-image-crop-picker/issues/191)
* watch out what to use in Embeeded Binaries in development and production builds, see [this github issue](https://github.com/ivpusic/react-native-image-crop-picker/issues/61)
* if multiple is true, cropping is not appreciated
* if multiple is false, get video from library can crash the picker

## Image

* GIF support   Image source ``require(‘./XXX.gif’)``
* text overlay image: use rgba(0,0,0, 0.6) to set the backgroundColor of overlay container

## react-native-orientation

* Orientation needs some code in AppDelegate.m for configuration, Need to configure it in AppDelegate.m to make it work
* Modal already has supportedOrientation, just unlockAllOrientations before showing Modal and lock after leaving Modal
* When using Dimensions to get window width and height, use them inside a function or component, instead of declaring them as constants

## Text and TextInput

* After Lei’s refactoring, ChapterShareView does not show text!  Fixed by removing View overhead for Text
* text input for title and description (value can be empty, placeholder can be non-empty)
* TextInput needs width and height!  [Grow height](http://stackoverflow.com/questions/33071950/how-would-i-grow-textinput-height-upon-text-wrapping)
* TextInput for better experience: save text to local database in the onChange function, dismiss keyboard (and text input) only when textFinished (i.e. user taps done or out-of-focus area)

## react-native-storybook

* If npm run storybook fails, check if it is already running, kill that process and rerun
* Image and static file support in React (and React Native) Storybook (see [this](http://getstorybook.io))

```javascript
import imageFile from ‘./static/image.png’
<img src={imageFile}> or <Image source={imageFile}>
```

No space or special symbols in the image name!

## Modal

* [Dismiss Modal by tapping backdrop](http://stackoverflow.com/questions/38311562/how-to-dismiss-modal-by-tapping-screen-in-reactnative)

## Keyboard

* After RN 0.27.0, use Keyboard instead of DeviceEventEmitter to register keyboardWillShow
* Mathematical calculation of content height is needed for scrolling upon keyboard show and dismiss

## react-native-swipelistview

* SwipeListView ``ref={(c) => {this._listView = c}}`` to get the ref of the SwipeListView, but to get the access to the underlying listview, need to use ``this._listView._listView``. Alternatively, according to the doc, one should use ``listViewRef={ ref => this._swipeListViewRef = ref }``, see [this](https://github.com/jemise111/react-native-swipe-list-view#listviewref)
* SwipeListView’s SwpieRow: recalculateHiddenLayout Enable hidden row onLayout calculations to run always.
By default, hidden row size calculations are only done on the first onLayout event for performance reasons. Passing true here will cause calculations to run on every onLayout event. You may want to do this if your rows' sizes can change. One case is a SwipeListView with rows of different heights and an options to delete rows.
* React Native 0.28 introduced new behavior when using flex: 1. The <SwipeRow> container <View> no longer has flex: 1 by default. If this is causing issues in your app you can maintain the old behavior by passing swipeRowStyle={{flex: 1}} to your <SwipeListView> or style={{flex: 1}} to your <SwipeRow>. See [this](https://github.com/jemise111/react-native-swipe-list-view#note-on-rn-028-and-flex-1)

## react-native-tabbar-navigator

* onChangeTab not get called because MainTabBar does not include onChange as a props!

## react-native-swiper

* loop MUST be false to enable 'index' property;  if use scroll responder, it will re-render and every slide will unmount and re-mount!
* Swiper components may not unmount!   That’s because it only renders different child components
* limit the number of components that are rendered (say, 3 adjacent).

## CameraRoll

* CameraRoll needs to be linked at first

## react-native-share

* WhatsApp url need correct encoding for sending text with chapter url

## PushNotificationsIOS

* PushNotificationsIOS fetchCompletionHandler, wrong instruction on the Facebook page, should [follow](http://stackoverflow.com/questions/36740143/silent-ios-push-notification-with-react-native-when-app-is-in-background)

# Download and Upload

Some generic refs

* [Download](https://www.raywenderlich.com/110458/nsurlsession-tutorial-getting-started)
* [Large file download](https://www.objc.io/issues/2-concurrency/common-background-practices/)

# Event Emitter and Listeners

* General guidance
  * success listeners removed in componentWillUnmount, which means some syncedAt are still null!  Move them to root view!
  * Use eventEmitter instead of refs to call certain functions of component

* 'sendDeviceEventWithName:body:' is deprecated: Subclass RCTEventEmitter instead
  * import "React/RCTEventEmitter.h" in project-swift-header.h
  * replace NSObject with RCTEventEmitter in the superclass area
  * implement override func supportedEvents
  * remove bridge variable, use self.sendEvent
  * use NativeEventEmitter instead of NativeModules to wrap the event-emit native module (see [this](http://facebook.github.io/react-native/releases/next/docs/native-modules-ios.html#sending-events-to-javascript))
  * Important: the event emitter and listener should be the same native module
  * Tip: pass a event function down to where the event is emitted
  * Note that any bridge other than the root view bridge will result in "Sending event with no listeners registered"
  * body (and constantsToExport) should be [String : Any]

# GraphQL and Apollo Client

Caveats

* Signup first time no token? all API calls return null!  Make sure client props is valid
* GraphQL validation needs definition of variables in the query level and mutation level

# Code Push

* Save config file to ~/.code-push.config
* Update upon app resume: ``Component = codePush({checkFrequency: codePush.CheckFrequency.ON_APP_RESUME, installMode: codePush.InstallMode.ON_NEXT_RESUME})(Component)``
* How to debug: use RCTSetLogThreshold(RCTLogLevelInfo) to enable logging in release version to debug

# Xcode

* objectForKey stringValue: if value is already NSString, NO NEED to call stringValue
* Project/General/Embedded Binaries to resolve the dyld error: dyld: Library not loaded
* create [GCD](http://stackoverflow.com/questions/37805885/how-to-create-dispatch-queue-in-swift-3) in Swift 3
* Swift-header may not be generated until all Swift files are successfully compiled. Try this: Comment out the#import line and all relevant lines till you can get "Build Succeeded", and then comment in the #importline and build again.
* Check stray unmatched opening brace somewhere for the error “SyntaxError: export declarations may only appear at top level”!!!
* RCTResponseSenderBlock correct [use](http://carminedimascio.com/2016/01/create-a-react-native-custom-swift-component/)
* ProcessMethodSignature [Error](http://stackoverflow.com/questions/39692230/got-is-not-a-recognized-objective-c-method-when-bridging-swift-to-react-native): Not a recognized Objective-C method, for Swift 3 project: first argument needs underscore
* In Swift 3, when defining closure, add @escaping
* How to set up multiple [schemes](https://zeemee.engineering/how-to-set-up-multiple-schemes-configurations-in-xcode-for-your-react-native-ios-app-7da4b5237966#.7nh5fyk50) & configurations in Xcode for your React Native iOS app

# Git

* use .gitignore to ignore xcuserdata

# Realm

* When we release app updates, any new entry in the schema should be optional
