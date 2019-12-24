---
title: CoreServicesUIAgent internals
categories:
  - Reverse Engineering
tags:
  - Assembly
  - macOS
  - x86
---

The recent release of macOS 10.15.2 had some additional updates to the Xprotect yara rules within it. After reviewing [what changed](https://github.com/knightsc/XProtect/commit/750e22c1bd870f8ee8f0f5ec9b6d410f0c90a0e3){: target="_blank"} in the yara rules I decided to dig a little deeper into how Xprotect gets called. Jonathan Levin's excellent book [MacOS and iOS Internals, Volume III: Security & Insecurity](https://www.amazon.com/MacOS-iOS-Internals-III-Insecurity/dp/0991055535/){: target="_blank"} briefly talks about Gatekeeper and Xprotect but didn't have the internals I was looking for. I ended up finding Patrick Wardle's excellent [presentation](https://www.virusbulletin.com/uploads/pdf/conference_slides/2015/Wardle-VB2015.pdf){: target="_blank"} from the 2015 Virus Bulletin Conference. His slide deck does a great job of explinaing the communication between `LaunchServices`, `CoreServicesUIAgent` and the `XprotectService`. It did, however, make me question what all does `CoreServicesUIAgent` do? This posts digs into the internals of `CoreServicesUIAgent` and documents its functionality.

# CoreServicesUIAgent

`CoreServicesUIAgent` has the responsibility of providing the occasional GUI to the end user when various other system frameworks need user input. The app bundle can be found in `/System/Library/CoreServices/CoreServicesUIAgent.app` and is installed as a LaunchAgent via the `/System/Library/LaunchAgents/com.apple.coreservices.uiagent.plist`. The LaunchAgent plist is short enough that I'll include the whole thing here:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>Label</key>
	<string>com.apple.coreservices.uiagent</string>
	<key>Program</key>
	<string>/System/Library/CoreServices/CoreServicesUIAgent.app/Contents/MacOS/CoreServicesUIAgent</string>
	<key>MachServices</key>
	<dict>
		<key>com.apple.coreservices.code-evaluation</key>
		<true/>
		<key>com.apple.coreservices.quarantine-resolver</key>
		<true/>
	</dict>
	<key>EnablePressuredExit</key>
	<true/>
	<key>EnableTransactions</key>
	<true/>
	<key>POSIXSpawnType</key>
	<string>Interactive</string>
</dict>
</plist>
```

When a user logs into the system, an instance of `CoreServicesUIAgent` is started and we can see from the plist that it has two Mach services that it provides. I'll start the exploration of the binary by looking at what it does at startup and then dive into each of the services.

## Initialization

If we look at the `CoreServicesUIAgent` binary the application delegate class is `CSUIController`. Looking at this class we can see what happens when `CoreServicesUIAgent` starts up.

* Create a `com.apple.coreservices.uiagent.active-handler-queue` dispatch queue
* Register a handful of message handlers (quarantine, launch error, file open, check access, etc)
* Register to listen for locale changes
* Create a new instance of `CSUICodeEvaluationController`, which in turn creates the `NSXPCListener` object for `com.apple.coreservices.code-evaluation`
* Call `xpc_connection_create_mach_service` for the `com.apple.coreservices.quarantine-resolver` service
* Calls the `CSUIController` classes `beginListeningForWindowChanges` method to listen for `NSWindow` lifecycle events

It's a bit odd that the `code-evaluation` service makes use of a `NSXPCListener` object while the `qaurantine-resolver` makes use of the C based XPC APIs. My guess is the `quarantine-resolver` code has been around for longer.

## com.apple.coreservices.quarantine-resolver

Searching for callers of this service we find that the primary caller is the `LaunchServices.framework`. Within the `LaunchServices.framework` there is a `__LSAgentGetConnection()` function that gets called in various other functions to retrieve an XPC connection to this service. I'll start by covering some of the general connection handling of this service.

### CSUIController handleIncomingXPCConnection

This handler gets set up during `CoreServicesUIAgent` initialization. It's responsible for accepting or denying incoming XPC connections. In general there are only a few basic checks. Keep in mind that `CoreServicesUIAgent` is a launch agent so it's running as the currently logged in user. It starts by calling `xpc_connection_get_audit_token` to get security information from the calling process. It then proceeds to check the euid and asid to ensure that it's the same logged in user trying to call the service.

There is alternatively an entitlement check for the `com.apple.private.coreservicesuiagent.allowedtouseCSUIFromOutsideSession` entitlement to see if a different user is allowed to call the service. As far as I could tell no application on the system has this entitlement.

### CSUIController handleIncomingXPCMessage

If the connection is accepted then the next step is to read and handle the XPC message itself. Every valid XPC message sent to this service has an `int64` value sent in the dictionary called `cmd` which lets the service know which message handler to hand off to.

At this point if an invalid `cmd` value is sent then the service calls `xpc_dictionary_create_reply` and sets an `int64` value with a name of `com.apple.coreservices.uiagent.status` and a negative error code.

Finally this method will call `scheduleHandlerForMessageType` to actually create and run the appropriate handler.

### CSUIController scheduleHandlerForMessageType

This method starts by calling `CSUIMessageHandler classForMessageType` which will loop through the registered message handlers and attempt to initialize the appropriate one for the `cmd` value sent in the XPC message. It then gets the audit token and checks whether the message handler allows callers that are sandboxed processes. Only some of the registered message handlers allow this. Finally, if permission checks pass, the method schedules a call to the message handlers `handleMessageWithCompletionHandler` method on a dispatch queue with the name `com.apple.coreservices.uiagent.background-message-queue`.

### Message Handlers

The message handlers make up the majority of the logic for this service. All the various message handlers get registered when `CoreServicesUIAgent` starts up and based on the `cmd` various handlers get called.

#### CSUIMessageHandler

All message handlers inherit from the `CSUIMessageHandler` base class. Most of the class logic is implemented in class methods that allow the registration and lookup of appropriate message handlers. But there are also default implementations of some of the message handler instance methods as well. For example `canBeUsedBySandboxedApplications` has a default return value of `NO` and only the message handlers that need to allow sandbox access override this method.

#### CSUIQuarantineMessageHandler

This handler is responsible for Gatekeeper and Xprotect checking. It's called from the `LaunchService.framework` `_LSOpenStuffCallLocal` function. As mentioned in the blog opening Patrick Wardle has a great [slide deck](https://www.virusbulletin.com/uploads/pdf/conference_slides/2015/Wardle-VB2015.pdf){: target="_blank"} that covers the details fairly well. Ultimately though this message handler delegates most of the checking off to the `GKQuarantineResolver` class. This class attempts to check and resolve items that are marked as quarantined. It makes use of the `Xprotect.framework` `XProtectAnalysis` class to call into the `XprotectService`.

Overall this message handler seems like a weird fit into this service. Most of the other services are really in `CoreServicesUIAgent` to present the occasional dialog to the end user. While this handler does show dialogs to the user if items are blocked it does also have the quarantine resolver logic. Some of this code almost seems like it should be in it's own service and then that service calls into this if it needs to present a dialog to the user.

The XPC message values can be seen below:

##### XPC Values

| Key | Type | Value |
|-----|------|-------|
| cmd | int64 | 0x1 |
| data | data | See Data Values below. |
| flags | uint64 | ? |
| firstLaunchPSN | uint64 | process serial number |


##### Data Values

| Key | Type | Value |
|-----|------|-------|
| LSQAllowUnsigned | NSNumber | YES or NO |
| LSQAppPSN | NSNumber | process serial number |
| LSQAppPath | NSString | Full path to the application to be evaluated |
| LSQAuthorization | NSData | A serialized AuthorizationRef object |
| LSQItemSandboxExtensions | NSDictionary | ? |
| LSQRiskCategory | NSString | Various risk categories. For example: "LSRiskCategoryUnsafeExecutable" |

#### LSLaunchErrorHandler

This message handler gets called from the `LaunchServices.framework` `_LSErrorDictPost` function. This function is called at various spots in the `LaunchServices.framework` when there's a problem trying to launch an application. The only thing this message handler does is present an error dialog to the user letting them know there was an issue launching the application.

The XPC message values can be seen below:

##### XPC Values

| Key | Type | Value |
|-----|------|-------|
| cmd | int64 | 0x2 |
| data | data | See Data Values below. |


##### Data Values

| Key | Type | Value |
|-----|------|-------|
| Action | NSString | Not sure what this is used for. `GURL` is one possible value |
| ErrorCode | NSNumber | The error code from trying to launch an application |
| AppPath | NSString | Full path to the application that couldn't be launched |
| AppMimimumSystemVersion | NSString | ? |
| AppMaximumSystemVersion | NSString | ? |

#### CSUILSOpenHandler

This is an interesting handler because it can actually be called from sandboxed application. It's called from the `LaunchService.framework` `_LSRemoteOpenCall invokeWithError` method. If the caller is sandboxed then the `sandbox_check_by_audit_token` method is called checking if the caller was granted `lsopen` privileges. Doing a quick search through sandbox profiles we can see the following applications that are allowed to call `lsopen`:

```
com.apple.AppSSOAgent.sb:(allow lsopen)
com.apple.CommerceKit.TransactionService.sb:(allow lsopen)
com.apple.ServicesUIAgent.sb:(allow lsopen)
com.apple.appstoreagent.sb:(allow lsopen)
com.apple.appstored.sb:(allow lsopen)
com.apple.assistantd.sb:(allow lsopen)
com.apple.commerce.sb:(allow lsopen)
com.apple.commerced.sb:(allow lsopen)
com.apple.iMessage.shared.sb:(allow lsopen)
com.apple.storeassetd.sb:(allow lsopen)
com.apple.storedownloadd.sb:(allow lsopen)
com.apple.storeuid.sb:(allow lsopen)
com.apple.studentd.sb:(allow lsopen)
com.apple.telephonyutilities.callservicesd.sb:(allow lsopen)
com.apple.touristd.sb:(allow lsopen)
fmfd.sb:(allow lsopen)
usernoted.sb:(allow lsopen)
```

This handler is interesting because normally from a non sandboxed application the caller can just use `LaunchServices.framework` directly to open another application. This handler seems to really just be here to provide this functionality to sandboxed applications. If the caller has the appropriate sandbox permissions then this message handler simply calls into the `_LSOpenCallInvokeXPC` function of `LaunchServices.framework`. It seems like for sandboxed applications `CoreServuceisUIAgent` provides a convienant launch agent process to put random code into.

The XPC message values can be seen below:

##### XPC Values

| Key | Type | Value |
|-----|------|-------|
| cmd | int64 | 0x3 |
| call | data | A serialized version of a `_LSRemoteOpenCall` instance. |

#### CSUICheckAccessHandler

This is another message handler that is callable from the sandbox and seems to only exist as a way to provide functionality to sandboxed processes. All this message handler does is call the `access` system call. When calling `access` the value `R_OK` is passed in as the `mode`. It then returns the value it gets back to the caller in a `flags` value. It's called from `_BindingBlueprint::checkUserAccessFlags` in `LaunchServices.framework`.

The XPC message values can be seen below:

##### XPC Values

| Key | Type | Value |
|-----|------|-------|
| cmd | int64 | 0x5 |
| path | string | The full path to the file to check |


#### CSUIGetDisplayNameHandler

Another handler that can be called from sandboxed applications. This one seems to be just for retrieving the proper localized name of a given application. It gets called from `_LaunchServices::URLPropertyProvider::prepareLocalizedNameValue` in the `LaunchServices.framework`.

The XPC message values can be seen below:

##### XPC Values

| Key | Type | Value |
|-----|------|-------|
| cmd | int64 | 0x7 |
| path | string | The path to an installed application |
| com.apple.coreservices.uiagent.localizations | xpc_object_t | ? |      

#### CSUIChangeDefaultHandlerHandler

At first glance this message handler seems like it might be interesting. It's called from `_LSSetSchemeHandler` in `LaunchServices.framework`. When a message is sent over to this handler it replies right away and then runs a code block. After digging a little deeper into this it really is only displaying a prompt to the end user. This message handler will call back into the `LaunchServices.framework` using the `LSDModifyService` class which in turn asks `lsd` to do the real work.

The XPC message values can be seen below:

##### XPC Values

| Key | Type | Value |
|-----|------|-------|
| cmd | int64 | 0x8|
| scheme | string | The scheme of the document handler that is being changed |
| bundleidentifier | string | The bundle id of the new document handler |              
| bundleversion | string | The version number of the bundle that will be the new document handler |              
| remove | bool | A bool indicating whether the handler is being removed or not. |              

#### CSUIRecommendSafariHandler

Another handler that can be called from the sandbox. This message handler appears to be what displays the prompt if Safari isn't the default browser. It's called from a block in `LSApplicationCheckin` in the case of a process that is a browser.

The XPC message values can be seen below:

##### XPC Values

| Key | Type | Value |
|-----|------|-------|
| cmd | int64 | 0x9|

## com.apple.coreservices.code-evaluation

This services makes use of the `NSXPC*` classes. It's only called from the `LaunchServices.framework`. The `LaunchServices.framework` has a `LSCodeEvaluationClientManager` class that is part of the `LSCodeEvaluation` class. `LSCodeEvaluationClientManager` calls into the remote service of `CoreServucesUIAgent` and makes use of the `LSCodeEvaluationServerProtocol` for the remote interface. In `CoreServicesUIAgent` the `CSUICodeEvaluationController` implements the `LSCodeEvaluationServerProtocol`. The protocol can be seen below:

```objc
@class LSCodeEvaluationInfo, NSString, NSUUID;

@protocol LSCodeEvaluationServerProtocol
- (void)showSecurityPreferencesAnchor:(NSString *)arg1;
- (void)showOriginForInfo:(LSCodeEvaluationInfo *)arg1;
- (void)ejectVolumeForInfo:(LSCodeEvaluationInfo *)arg1;
- (void)moveItemToTrashForInfo:(LSCodeEvaluationInfo *)arg1;
- (void)closeProgressForInfo:(LSCodeEvaluationInfo *)arg1;
- (void)updateProgressForInfo:(LSCodeEvaluationInfo *)arg1;
- (void)startProgressForInfo:(LSCodeEvaluationInfo *)arg1;
- (void)registerWithLaunchServicesForInfo:(LSCodeEvaluationInfo *)arg1;
- (void)updateQuarantineFlagsForInfo:(LSCodeEvaluationInfo *)arg1 setFlags:(int)arg2 clearFlags:(int)arg3 reply:(void (^)(void))arg4;
- (void)presentPromptOfType:(long long)arg1 options:(long long)arg2 info:(LSCodeEvaluationInfo *)arg3 identifier:(NSUUID *)arg4;
@end
```

I'm not entirely clear how and when the `LaunchServices.framework` makes use of this remote service but for the most part this service is really just presenting dialogs to the end user. Methods in the protocol above that seem to take an actual action end up calling back into `LaunchServices.framework` to do any of the real work.

# Conclusion

This proved to be an interesting deep dive into the internals of a system launch agent that probably most of us take for granted. Additionally, there really seem to three types of main functionality that `CoreServicesUIAgent` handles:

* Coordinating and running quarantine malware checks
* Providing random functionality to sandboxed processes
* Displaying dialogs to the logged in user

Displaying dialogs is clearly the majority of what the launch agent does. Historically I would guess this was also the only thing in `CoreServicesUIAgent`. Over time it seems like like the launch agent provided a convienant point to add additional functionality that should be running as the logged in user. All of the above internals comes from macOS 10.15.2. It would be interesting to go back over the last couple major versions of macOS and see how the internal mach services have changed or grown over time. Finally if you want to play around with sending some XPC messages to `CoreServicesUIAgent` you can find a small command line program I used for testing [here](https://gist.github.com/knightsc/a22967814423934ef4e7a3f3c235dd11){: target="_blank"}.
