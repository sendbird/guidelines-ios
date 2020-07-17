# Make Local Calls with CallKit

This page guides you through the process of developing VoIP apps by using Sendbird Calls framework and Apple’s CallKit framework. You’ll start by developing a simple project that allows you to make local calls using CallKit.

Each section provides entire codes of the file. You can just copy and paste the code into the appropriate files. Provided code may not be the only correct implementations. You can customize them to suit your needs by thoroughly reviewing the following steps. These steps will help you understand more about CallKit.

You can download the complete project file here.

### Estimated Time

30 min

## Section1: The basic configuration of CallKit

### Step1

To develop VoIP app service, you need VoIP certificate for the app. Go to Apple Developer page and sign in.

### Step2

Go to Certificate, Identifiers & Profiles > Certificates > Create a New Certificate. You can find VoIP Services Certificate under Services section. Create VoIP service certificate.

### Step3

Go to Target > Signing & Capabilities. Add Background Modes and enable Voice over IP. This will create an .entitlements file and appropriate permissions that allow you to use VoIP services. If you don’t enable Voice over IP, CallKit error code 1 will occur.

## Section2: Design CallKit UI

### Step1

To configure localized information for CallKit, create a file named `CXProviderConfiguration.extension.swift`. 

#### CXProvider
> `CXProvider` is an object that represents a telephony provider. `CXProvider` is initialized with `CXProviderConfiguration`. VoIP apps should create only one instance of `CXProvider` per app and use it globally. For more information, see [Apple Developer Docs - CXProvider](https://developer.apple.com/documentation/callkit/cxprovider)

Each provider can specify an object conforming to the `CXProviderDelegate` protocol to respond to events, such as call starting, call being put on hold, or provider’s audio session being activated.

### Step2 

```swift
extension CXProviderConfiguration {
    static var custom: CXProviderConfiguration {
        // 1
        let configuration = CXProviderConfiguration(localizedName: "Homing Pigeon")

        // 2
        // Native call log shows video icon if it was video call.
        configuration.supportsVideo = true

        // Support generic type to handle *User ID*
        configuration.supportedHandleTypes = [.generic]

        // Icon image forwarding to app in CallKit View
        if let iconImage = UIImage(named: "App Icon") {
            configuration.iconTemplateImageData = iconImage.pngData()
        }

        return configuration
    }
}
```

> A `CXProviderConfiguration` object controls the native call UI for incoming and outgoing calls, including the localized name of the provider, ringtone to be played for incoming calls, and the icon to be displayed during calls. For more information, see [Apple Developer Docs - CXProviderConfiguration](https://developer.apple.com/documentation/callkit/cxproviderconfiguration)

1. Initialize `CXProviderConfiguration` object with a localized name. This name will appear in the call view when your users receive a call through CallKit. Use appropriate naming, such as your app service name, as the localized name. In this case, I named it “Homing Pigeon”

2. Let’s configure the user interface and its capabilities. In this step, I’m going to set `supportsVideo` , `supportedHandleTypes` and `iconTemplateImageData`. If you want to customize CallKit more , see the table below. You can also refer to [Apple Developer Document - CXProviderConfiguration](https://developer.apple.com/documentation/callkit/cxproviderconfiguration)

| name | description | default value |
| --- | --- | --- |
| ringtoneSound | The name of the sound resource in the app bundle to be used for the provider ringtone. | | 
| iconTemplateImageData | The PNG data for the icon image to be displayed for the provider. | |
| maximumCallGroups | The maximum number of call groups | 2 |
| maximumCallsPerCallGroup | The maximum number of calls per call group. | 5 |
| supportedHandleTypes | The supported handle types. | [ ] |
| supportsVideo | A Boolean value that indicates whether the provider supports video in addition to audio. | false |

#### SupportVideo
```swift
configuration.supportsVideo = true
```
> Boolean value that indicates whether the call supports video capability in addition to audio. By default, it’s set to `false`. Apple Document: [supportsVideo](https://developer.apple.com/documentation/callkit/cxproviderconfiguration/1779574-supportsvideo)

If your service provides video call, set `supportsVideo` to `true` . If your service does not provide video call, skip this setting.

#### supportedHandleTypes
```swift
configuration.supportedHandleTypes = [.generic]
```
> Types of call provider that you want to handle. See [CXHandle.HandleType](https://developer.apple.com/documentation/callkit/cxhandle/handletype)

`CXHandle` refers to how your users are identified in each call. Three possible types for handle are `.phoneNumber`, `.email`, and `.generic`. Depending on the service you provide and how you manage your users, you may choose different options. If the users are identified by their phone number or their email address, choose `.phoneNumber` or `.email`.  However, if it’s based on some random `UUID` value or other unspecified value, use `.generic`. `.generic` is an unspecified `String` value that can be used more flexibly.

#### iconTemplateImageData
```swift
if let iconImage = UIImage(named: "App Icon") {
    configuration.iconTemplateImageData = iconImage.pngData()
}
```
> The PNG data for the icon image to be displayed for the provider.
> 
> The icon image should be a square image with a side length of 40 points. The alpha channel of the image is used to create a white image mask, which is used in the system native in-call UI for the button which takes the user from this system UI to the 3rd-party app.
> 
> See [iconTemplateImageData](https://developer.apple.com/documentation/callkit/cxproviderconfiguration/2274376-icontemplateimagedata)

Set `.iconTemplateImageData` to the icon image that will be displayed next to the localized name on the CallKit screen. Assign `.pngData()` to your app icon.

## Section3: Request CallKit action

CallKit provides many call-related features such as dialing, ending, muting, holding, etc. Each of these features should be executed by appropriate CallKit actions called `CXCallAction`. These actions are called from a `CXCallController` object, which uses `CXTransaction` objects to execute each `CXCallAction`. In order to control CallKit, you must create corresponding `CXCallActions` and execute them by requesting transaction with `CXTransaction`. 

There are 3 Steps to send a request to CallKit
1. Create `CXCallAction` object
2. Create `CXTransaction` object
3. Request the `CXTransaction` object via `CXCallController`

| Name | Description | Reference |
| --- | --- | --- |
| CXCallAction | Telephony actions, such as start call, end call, mute call, hold call, associated with a call object |  [ Developer - CXCallAction](https://developer.apple.com/documentation/callkit/cxcallaction) |
| CXTransaction | An object that contains zero or more action objects to be performed by a call controller. | [ Developer - CXTransaction](https://developer.apple.com/documentation/callkit/cxtransaction) |
| CXCallController | An object interacts with calls by performing actions and observing calls. | [ Developer - CXCallController](https://developer.apple.com/documentation/callkit/cxtransaction) |

### Step1. Transaction
```swift
// Allow to request for actions
let callController = CXCallController()

// Request transaction
private func requestTransaction(with action: CXCallAction, completionHandler: (NSError? -> Void)?) {
    let transaction = CXTransaction(action: action)
    callController.request(transaction) { error in
        completionHandler?(error as NSError?)
    }
}
```
Add `CXCallController` property and other method named `requestTransaction(with:completionHandler:)`.  The method creates `CXTransaction` with `CXCallAction` and request the transaction via `callController`. You always have to call this method after creating `CXCallAction` object.

### Step2. Call Actions

#### Start Call Action
Let’s implement a method for `CXStartCallAction`.  This action represents the start of a call. If the action was requested successfully, a corresponding `CXProviderDelegate.provider(_:perform:)` event will be called.
```swift
func startCall(with uuid: UUID, calleeID: String, hasVideo: Bool, completionHandler: ((NSError?) -> Void)? = nil) {
    // 1
    let handle = CXHandle(type: .generic, value: calleeID)    
    let startCallAction = CXStartCallAction(call: uuid, handle: handle)
    
    // 2
    startCallAction.isVideo = hasVideo
    
    // 3
    self.requestTransaction(with: startCallAction, completionHandler: completionHandler)
}
```
1. You have to create a `CXHandle` object associated with the call that will be used to identify the users involved with the call. This object will be included in `CXStartCallAction` along with a `UUID`. 
2. If the call has video, set `.isVideo` to `true`.
```swift
startCallAction.isVideo = hasVideo
```
3. As I mentioned at **Step1**, don’t forget to call `requestTransaction(with:completionHandler:)` method after creating `CXStartCallAction` object.

#### End Call Action

Let’s implement another method for `CXEndCallAction`. This action represents that the call was ended.  If the action was requested successfully, a corresponding `CXProviderDelegate.provider(_:perform:)` event will be called. `CXEndCallAction` only requires the `UUID` of the call. Create a `CXEndCallAction` object with the `UUID`.
```swift
func endCall(with uuid: UUID, completionHandler: ((NSError?) -> Void)? = nil) {
    let endCallAction = CXEndCallAction(call: uuid)
    self.requestTransaction(with: endCallAction, completionHandler: completionHandler)
}
```
#### Other Call Actions
Other `CXCallActions` can be implemented as same as `CXStartCallAction` and `CXEndCallAction` . Here is the list of other call actions.

| Call Action | Description | Corresponding Event |
| --- | --- | --- |
| [CXAnswerCallAction](https://developer.apple.com/documentation/callkit/cxanswercallaction) |  Answers an incoming call. | [CXProviderDelegate.provider(_:perform:)](https://developer.apple.com/documentation/callkit/cxproviderdelegate/1648270-provider) |
| [CXSetHeldCallAction](https://developer.apple.com/documentation/callkit/cxsetheldcallaction) | Places a call on hold or removes a call from hold. | [CXProviderDelegate.provider(_:perform:)](https://developer.apple.com/documentation/callkit/cxproviderdelegate/1648256-provider) |
| [CXSetMutedCallAction](https://developer.apple.com/documentation/callkit/cxsetmutedcallaction) | Mutes or unmutes a call. | [CXProviderDelegate.provider(_:perform:)](https://developer.apple.com/documentation/callkit/cxproviderdelegate/1648269-provider) |
| [CXSetGroupCallAction](https://developer.apple.com/documentation/callkit/cxsetgroupcallaction) | Groups a call with another call or removes a call from a group. | [CXProviderDelegate.provider(_:perform:)](https://developer.apple.com/documentation/callkit/cxproviderdelegate/1648269-provider) |

## Section4. Manage Call
To easily manage manage `CXCallController` and call IDs, you may want to create a call manager which must be accessible from anywhere. The call manager will store and manage `UUID`s of the ongoing calls to handle call events. 

> **NOTE** 
> You can also use `CXCallController.callObserver.calls` property that manages a list of active calls(including ended calls) and observes call changes. Each call is a `CXCall` object that represents a call in CallKit. By checking the `hasEnded` attribute, you can handle ongoing calls.
> 
> For more information, see [Apple Developer Document - CallObserver](https://developer.apple.com/documentation/callkit/cxcallobserver) and [Apple Developer Document - CXCall](https://developer.apple.com/documentation/callkit/cxcall)
```swift
import CallKit

class CallManager {
    // 1
    static let shared = CallManager()

    let callController = CXCallController()

    // 2
    private(set) var callIDs: [UUID] = []

    // MARK: Call Management
    func containsCall(uuid: UUID) -> Bool {
        return CallManager.shared.callIDs.contains(where: { $0 == uuid })
    }

    func addCall(uuid: UUID) {
        self.callIDs.append(uuid)
    }

    func removeCall(uuid: UUID) {
        self.callIDs.removeAll { $0 == uuid }
    }

    func removeAllCalls() {
        self.callIDs.removeAll()
    }
}
```

Create a new class named `CallManager`. Then, add a shared static instance to access it from everywhere. (You may choose to use other patterns than singleton)
```swit
static let shared = CallManager() // singleton
```
> **Information**
> If you want to know more about this pattern, see [Managing a shared resource using a singleton](https://developer.apple.com/documentation/swift/cocoa_design_patterns/managing_a_shared_resource_using_a_singleton)

2. Add `callIDs` property with a type of `[UUID]` and add methods for managing `callIDs` : `addCall(uuid:)`, `removeCall(uuid:)` and `removeAllCalls()`
```swift
private(set) var callIDs: [UUID] = []

func containsCall(uuid: UUID) -> Bool { ... }

func addCall(uuid: UUID) { ... }

func removeCall(uuid: UUID) { ... }

func removeAllCalls() { ... }
```

## Section5: Handling CallKit Events

To report new incoming calls or respond to new CallKit actions, you have to create a `CXProvider` object with `CXProviderConfiguration` that was created at [**Section2**](##Section2). You can also handle `CallKit` events of the call via `CXProviderDelegate`.

```swift
// ProviderDelegate.swift
class ProviderDelegate: NSObject {

    // 2
    private let provider: CXProvider

    override init() {
        provider = CXProvider(configuration: CXProviderConfiguration.custom)

        super.init()

        // If the queue is `nil`, delegate will run on the main thread.
        provider.setDelegate(self, queue: nil)
    }

    // 3
    func reportIncomingCall(uuid: UUID, callerID: String, hasVideo: Bool, completionHandler: ((NSError?) -> Void)? = nil) {

        // Update call based on DirectCall object
        let update = CXCallUpdate()

        // 4. Informations for iPhone local call log
        let callerID = call.caller?.userId ?? "Unknown"
        update.remoteHandle = CXHandle(type: .generic, value: callerID)
        update.localizedCallerName = callerID
        update.hasVideo = hasVideo

        // 5. Report new incoming call and add it to `callManager.calls`
        provider.reportNewIncomingCall(with: uuid, update: update) { error in
            guard error == nil else {
                completionHandler?(error as NSError?)
                return
            }

            // Add call to call manager
            CallManager.shared.addCall(uuid: uuid)
        }
    }

    // 6
    func connectedCall(uuid: UUID, startedAt: Int64) {
        let connectedAt = Date(timeIntervalSince1970: Double(startedAt)/1000)
        self.provider.reportOutgoingCall(with: uuid, connectedAt: connectedAt)
    }

    // 7
    func endCall(uuid: UUID, endedAt: Date, reason: CXCallEndedReason) {
        self.provider.reportCall(with: uuid, endedAt: endedAt, reason: reason)
    }
}

// 1
extension ProviderDelegate: CXProviderDelegate {
    func providerDidReset(_ provider: CXProvider) { }
}
```
1. Import `CallKit` and create a `ProviderDelegate` class with `NSObject` and `CXProviderDelegate` conformance.
2. Add two properties: `callManager` and `provider`. The `callManager` is the `CallManager` class that you created in [**Section 3**](#Section-3). The `provider` reports actions for CallKit. When you initialize a provider , use `CXProviderConfiguration.custom` that you already created at [**Section 2**](#Section-2).

```swift
private let provider: CXProvider

override init() {
    provider = CXProvider(configuration: CXProviderConfiguration.custom)

    super.init()

    // If the queue is `nil`, delegate will run on the main thread.
    provider.setDelegate(self, queue: nil)
}
```
3. Let’s report a new incoming call. To report a new incoming call, you need to create a `CXCallUpdate` instance with the relevant information about the incoming call as well as the `CXHandle` that identifies the users involved in the call.
4. To make your calls richer, you can customize the `CXHandle` and `CXCallUpdate` instances. If the call has video, set `hasVideo` to `true`. Upper iPhone call log is based on `CXHandle` object. 
5. After reporting a new incoming call, you have to add it to `CallManager.shared.calls` by using the `addCall(uuid:)` method that we added earlier.
6. CallKit keeps track of the connected time of the call and the end time of the call by listening to appropriate CallKit events. 
To tell the CallKit that the call was connected, call `reportOutgoingCall(with:connectedAt:)`. This initiates the call duration elapsing, and informs the starting point of the call that is displayed in the call log of the iPhone app. 
7. To tell the Callkit that the call was ended, call `reportCall(with:endedAt:reason:)`. This informs the end point of the call that will be displayed in the call log of the iPhone app as well.

## Section6: Handle CXCallAction event 

### (Interaction with CallKit UI)

When the provider performs `CXCallActions`, corresponding `CXProviderDelegate` methods can be called. In order to properly respond to the users’ actions, you have to implement appropriate Sendbird Calls actions in the method.

> **Important**
> Don’t forget to execute `action.fulfill()` before the method is ended.

| method | Description & What to do in here? | Sendbird Calls method |
| --- | --- | --- |
| func provider(CXProvider, perform: CXStartCallAction) | You can get call object from `CXStartCallAction` object. Add its call ID to `callManger.callIDs`. For iPhone local call logs(Recents), you may call `provider.reportOutgoingCall(with:startedConnectingAt:)`. | If you want to handle `DirectCall` object, use `SendBirdCall.getCall(forUUID:)` |
| func provider(CXProvider, perform: CXAnswerCallAction) | This method called when the user tapped mute button on CallKit UI screen. Invoke accepting action in Sendbird Calls. | `DirectCall.accept(with:)` |
| func provider(CXProvider, perform: CXEndCallAction) | This method called when the user tapped end button on CallKit UI screen. Invoke ending action in Sendbird Calls. | `DirectCall.end()` |
| func provider(CXProvider, perform: CXSetMutedCallAction) | This method called when the user tapped mute button on CallKit UI screen. Invoke muting / unmuting action in Sendbird Calls | `DirectCall.muteMicrophone()` and `DirectCall.unmuteMicrophone()` | 
| func provider(CXProvider, timedOutPerforming: CXAction) | This method called when the provider performs the specified action times out. (Optional) You may invoke ending action. | | 

For more information about `CXProviderDelegate` methods, refer to [Apple Developer Document - CXProviderDelegate](https://developer.apple.com/documentation/callkit/cxproviderdelegate)

```swift
// ProviderDelegate.swift
extension ProviderDelegate: CXProviderDelegate {
    func providerDidReset(_ provider: CXProvider) {
        // Stop audio
        // End all calls because they are no longer valid
        // Remove all calls from the app's list of call

        CallManager.shared.removeAllCalls()
    }

    func provider(_ provider: CXProvider, perform action: CXStartCallAction) {
        // Get call object
        // Configure audio session
        // Add call to  `callManger.callIDs`.
        // Report connection started

        action.fulfill()
    }

    func provider(_ provider: CXProvider, perform action: CXAnswerCallAction) {
        // Configure audio session
        // Accept call
        // Notify incoming call accepted

        action.fulfill()
    }

    func provider(_ provider: CXProvider, perform action: CXEndCallAction) {
        // Mute the call
        // End the call

        action.fulfill()

        // Remove the ended call from `callManager.callIDs`.
        CallManager.shared.removeCall(uuid: action.callUUID)
    }

    func provider(_ provider: CXProvider, perform action: CXSetHeldCallAction) {
        // update holding state.
        // Mute the call when it's on hold.
        // Stop the video when it's a video call.

        action.fulfill()
    }

    func provider(_ provider: CXProvider, perform action: CXSetMutedCallAction) {
        // stop / start audio

        action.fulfill()
    }

    func provider(_ provider: CXProvider, didActivate audioSession: AVAudioSession) {
        // Start audio
    }

    func provider(_ provider: CXProvider, didDeactivate audioSession: AVAudioSession) {
        // Restart any non-call related audio now that the app's audio session has been
        // de-activated after having its priority restored to normal.
    }
}
```

## Section7: Interaction with UI

Now, we can start and end calls with CallKit using its default view. Next, let’s try to use a custom UI with CallKit.  For the sake of clarity, I’ll skip creating related storyboard files and `ViewController` files. Just suppose that there is one text field for entering the remote user’s ID, one button for making an outgoing call, another button for receiving an incoming call, and the last button for ending the call.

```swift
// ViewController.swift
import UIKit

class ViewController: UIViewController {
    let providerDelegate = ProviderDelegate()

    // UUID of ongoing call
    var callID: UUID?

    // 1
    @IBAction func didTapOutgoingCall() {
        guard let calleeID = userIDTextField.text?.trimmingCharacters(in: .whitespaces) else { return }
        guard !calleeID.isEmpty else { return }
        let uuid = UUID()
        self.callID = uuid

        CallManager.shared.startCall(with: uuid, calleeID: calleeID, hasVideo: false) { error in
            // ...
        }
    }

    // 2
    @IBAction func didTapEnd() {
        guard let callID = self.callID else { return }
        CallManager.shared.endCall(with: callID) { error in
            guard error == nil else { return }
        }
        self.callID = nil
    }

    // 3
    @IBAction func didTapIncomingCall() {
        guard let callerID = userIDTextField.text?.trimmingCharacters(in: .whitespaces) else { return }
        guard !callerID.isEmpty else { return }
        let uuid = UUID()
        self.callID = uuid

        providerDelegate.reportIncomingCall(uuid: uuid, callerID: callerID, hasVideo: false) { error in
            // ...
        }
    }
}
```
1. Let’s make an outgoing call. Because the user is initiating a call, you have to create a request for the call. This action requires callee’s `user ID` and unique `UUID` of the call.
2. Let’s implement the action for the end button. This action will end the call based on the `callID`.
3. Let’s try to answer an incoming audio call. To do this, we have to simulate an incoming audio call. Because CallKit is not aware of the incoming call, you have to report to the CallKit about the incoming call. This action requires the caller's user ID and unique `UUID` of the call. Currently, because the incoming call is made locally, you will use randomly generated `UUID()` instead of a real call’s `UUID`. If you want to test incoming video calls, assign the value of `hasVideo` parameter as `true`.

