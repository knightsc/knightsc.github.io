---
title: Apple ID authentication
categories:
  - Reverse Engineering
tags:
  - macOS
---

Since iTunes removed the ability to download and install iPhone applications I've been using Apple Configurator 2. I wanted to understand more about how it logged in and showed me a list of purchased apps and downloaded IPAs. Ideally I'd like to automate this. So I started tracking things down from the authentication step. Since most of this is private APIs there's no Apple documentation and we needed to do some reversing.

adid first appeared in 10.13

AuthKit first appeared in 10.11

10.11 Didn't have an AKADIProxy but still had the code and was still using indirect jmp/calls and obfuscation

# Call trace from signing in Apple Configurator

Apple Configurator 2 - Sign In
[ACUApplicationDelegate signIn]
|- [ACUStoreController signIn]
StoreFoundation
  |-  proxy = [[ISServiceProxy alloc] initWithStoreClient: [CKStoreClient sharedInstance]]
  |- ISAccountService *accountService = [proxy accountServiceWithErrorHandler]
    |- [accountService signInWithContext:replyBlock:] // old versions of the framework
CommerceKit
    |- [CKAccountStore sharedAccountStore] signInWithSuggesstedAppleId
      |- [CkAuthenticationContext alloc] initWithStoreClient] authenticateWithDialog
AuthKit
        |- AKAppleIDAuthenticationController authenticateWithContext
XPC - com.apple.ak.auth.xpc (akd)
         |- AKAppleIDAuthenticationService authenticateWithContext
         |- AKAppleIDAuthenticationService _authenticateWithContext
        |- AKAppleIDAuthenticationService _performAuthenticationWithContext
        |- AKAppleIDAuthenticationService  _performSRPAuthenticationWithContext:completion:
        |- AKSRPOperation performWithURL:SRPContext
AppleIDAuthSupport
           |- AppleIDAuthSupportAuthenticate // Does the actual SRP auth

Bag service
https://gsa.apple.com/grandslam/GsService2/lookup

/* @class AKURLBag */
-(void *)basicAuthURL {
    rax = [self _urlAtKey:@"gsService"];
    return rax;
}


https://bbs.pediy.com/thread-247744-1.htm

/* @class AKAppleIDAuthenticationService */
[AKAppleIDAuthenticationService _performSRPAuthenticationWithContext:completion:]

/* @class AKAppleIDAuthenticationService */
-(void)_performSRPAuthenticationWithContext:(void *)arg2 completion:(void *)arg3 {

AKSRPOperation

AKSRPContext

0x0000000104481000

000000010002bc41

stateServerNeg1
stateClientNeg1
stateInvalid
stateDone


https://developer.apple.com/bug-reporting/profiles-and-logs/

AuthKit

https://blog.elcomsoft.com/2017/11/icloud-authentication-tokens-inside-out/#more-4445

# Enable logging

sudo log config --mode "level:debug" --subsystem com.apple.authkit
sudo log config --status --subsystem com.apple.authkit
sudo log stream --debug --predicate 'subsystem == "com.apple.authkit"'

https://developer.apple.com/bug-reporting/profiles-and-logs/

sudo log collect --debug --predicate 'subsystem == "com.apple.authkit"'

sudo log collect --output ~/Desktop/autokit.logarchive --start "2019-04-30 09:06:00"


# Get list of service URLs

KeyBagURL

https://gsa.apple.com/grandslam/GsService2/lookup 

{
    "X-Apple-I-Locale" = "en_US";
    "X-Apple-I-TimeZone" = "America/New_York";
    "X-Apple-I-TimeZone-Offset" = "-14400";
    "X-MMe-Client-Info" = "<MacBookPro14,3> <Mac OS X;10.14.4;18E226> <com.apple.AuthKit/1 ((null)/1.0)>";
    "X-MMe-Country" = US;
    "X-Mme-Device-Id" = "164D1BC8-A71B-1F58-0C20-1BF98DB4F27D";
}

# Get Anisette data

akd AKNativeAnisetteService

createProvisioningStartURLRequest

https://gsa.apple.com/grandslam/MidService/startMachineProvisioning

{
    "X-Apple-Client-App-Name" = "Apple Configurat";
    "X-Apple-I-Client-Time" = "2019-04-29T19:07:59Z";
    "X-Apple-I-MD-LU" = 74395C347FB810CAFBF0589D426A7DDAB71B19D75CFD69BC664EDD89ABC6F5DF;
    "X-Apple-I-MLB" = C018025031BHYRG1T;
    "X-Apple-I-ROM" = 4867f09f6ec5;
    "X-Apple-I-SRL-NO" = C02W34C3JUN9;
    "X-MMe-Client-Info" = "<MacBookPro14,3> <Mac OS X;10.14;18A391> <com.apple.AuthKit/1 (com.apple.akd/1.0)>";
    "X-Mme-Device-Id" = "164D1BC8-A71B-1F58-0C20-1BF98DB4F27D";
}

breakpoint set -r '\[AKADIProxy .*\]$'

# Check Apple ID


[  0] 42F02201-B614-317C-B345-E7B4AFF3FAFA 0x0000000104481000 /System/Library/PrivateFrameworks/AuthKit.framework/Versions/A/Support/akd 

 -[AKNativeAnisetteService fetchAnisetteDataAndProvisionIfNecessary:withCompletion:]:

GET

<NSMutableURLRequest: 0x7ff74dd8c430> { URL: https://gsa.apple.com/grandslam/GsService2/fetchAuthMode }

{
    "X-Apple-Client-App-Name" = "Apple Configurat";
    "X-Apple-I-AppleID" = "sdgsdg@sdfdf.com";
    "X-Apple-I-DeviceUserMode" = 0;
    "X-Apple-I-MD" = "BBBBBQAAABDkSxy8gO/F7nXlhZZYrXIYAAAAAQ==";
    "X-Apple-I-MD-LU" = 74395C347FB810CAFBF0589D426A7DDAB71B19D75CFD69BC664EDD89ABC6F5DF;
    "X-Apple-I-MD-M" = "Opo6kUlECGf1e+2mU6DK0VJ2GIvMuUd8LTWtLELZw5h6EX1xoCG81GSNQcH3Ha6MTXlLUBuZ4hGev+Lv";
    "X-Apple-I-MD-RINFO" = 17106170;
    "X-MMe-Client-Info" = "<MacBookPro14,3> <Mac OS X;10.14;18A391> <com.apple.AuthKit/1 (com.apple.akd/1.0)>";
}

Response says if passwordless auth is allowed otherwise prompt for password

# SRP part 1

Request 1

* thread #4, queue = 'AKUIWorkQueue', stop reason = breakpoint 1.1
  * frame #0: 0x00007fff2d135bb5 CFNetwork`-[NSURLSession dataTaskWithRequest:completionHandler:]
    frame #1: 0x00007fff3ec0767b AppleIDAuthSupport`-[AIASSession requestWithURL:data:clientInfo:proxiedClientInfo:companionClientInfo:] + 280
    frame #2: 0x00007fff3ec08615 AppleIDAuthSupport`SendRequestAndCreateResponse + 533
    frame #3: 0x00007fff3ec08366 AppleIDAuthSupport`AppleIDAuthSupportAuthenticate + 129
    frame #4: 0x00000001044d3a7f akd`___lldb_unnamed_symbol1460$$akd + 58
    frame #5: 0x00000001044af5f4 akd`___lldb_unnamed_symbol757$$akd + 817
    frame #6: 0x00000001044ae2e0 akd`___lldb_unnamed_symbol735$$akd + 290
    frame #7: 0x00000001044ab838 akd`___lldb_unnamed_symbol694$$akd + 948
    frame #8: 0x00000001044ad1e5 akd`___lldb_unnamed_symbol725$$akd + 224
    frame #9: 0x00007fff5b2bbd4f libdispatch.dylib`_dispatch_call_block_and_release + 12
    frame #10: 0x00007fff5b2bcdcb libdispatch.dylib`_dispatch_client_callout + 8
    frame #11: 0x00007fff5b2bf5d8 libdispatch.dylib`_dispatch_continuation_pop + 427
    frame #12: 0x00007fff5b2bec7a libdispatch.dylib`_dispatch_async_redirect_invoke + 718
    frame #13: 0x00007fff5b2cad1a libdispatch.dylib`_dispatch_root_queue_drain + 325
    frame #14: 0x00007fff5b2cb4b1 libdispatch.dylib`_dispatch_worker_thread2 + 90
    frame #15: 0x00007fff5b4fc6ee libsystem_pthread.dylib`_pthread_wqthread + 619
    frame #16: 0x00007fff5b4fc415 libsystem_pthread.dylib`start_wqthread + 13

[  0] 42F02201-B614-317C-B345-E7B4AFF3FAFA 0x0000000104481000 /System/Library/PrivateFrameworks/AuthKit.framework/Versions/A/Support/akd 
[  1] D6387150-2FB8-3066-868D-72E1B1C43982 0x000000010c992000 /usr/lib/dyld 
[  2] E9F2FF73-22D0-35B5-BD2C-9DD8FDB12BCC 0x00007fff4d8ce000 /System/Library/PrivateFrameworks/MobileKeyBag.framework/Versions/A/MobileKeyBag 
[  3] 5362D9AD-A2AE-3436-97CE-C353124504E5 0x00007fff3ec05000 /System/Library/PrivateFrameworks/AppleIDAuthSupport.framework/Versions/A/AppleIDAuthSupport 
[  4] ED375339-69F6-34AE-825D-F16BF0618E3E 0x00007fff3f3be000 /System/Library/PrivateFrameworks/AuthKit.framework/Versions/A/AuthKit 
[  5] BE71A0AD-7ED9-3B2A-BF7D-DA7841640FE0 0x00007fff4151c000 /System/Library/PrivateFrameworks/CommonUtilities.framework/Versions/A/CommonUtilities 


(lldb) po [$rdx allHTTPHeaderFields]
{
    "Content-Type" = "text/x-xml-plist";
    "X-MMe-Client-Info" = "<MacBookPro14,3> <Mac OS X;10.14;18A391> <com.apple.AuthKit/1 (com.apple.akd/1.0)>";
}

# SRP part 2

Request 2

URL: https://gsa.apple.com/grandslam/GsService2

 frame #0: 0x00007fff2d135bb5 CFNetwork`-[NSURLSession dataTaskWithRequest:completionHandler:]
    frame #1: 0x00007fff3ec0767b AppleIDAuthSupport`-[AIASSession requestWithURL:data:clientInfo:proxiedClientInfo:companionClientInfo:] + 280
    frame #2: 0x00007fff3ec08615 AppleIDAuthSupport`SendRequestAndCreateResponse + 533
    frame #3: 0x00007fff3ec08366 AppleIDAuthSupport`AppleIDAuthSupportAuthenticate + 129
    frame #4: 0x00000001044d3a7f akd`___lldb_unnamed_symbol1460$$akd + 58
    frame #5: 0x00000001044af5f4 akd`___lldb_unnamed_symbol757$$akd + 817
    frame #6: 0x00000001044ae2e0 akd`___lldb_unnamed_symbol735$$akd + 290
    frame #7: 0x00000001044ab838 akd`___lldb_unnamed_symbol694$$akd + 948
    frame #8: 0x00000001044ad1e5 akd`___lldb_unnamed_symbol725$$akd + 224
    frame #9: 0x00007fff5b2bbd4f libdispatch.dylib`_dispatch_call_block_and_release + 12
    frame #10: 0x00007fff5b2bcdcb libdispatch.dylib`_dispatch_client_callout + 8
    frame #11: 0x00007fff5b2bf5d8 libdispatch.dylib`_dispatch_continuation_pop + 427
    frame #12: 0x00007fff5b2bec7a libdispatch.dylib`_dispatch_async_redirect_invoke + 718
    frame #13: 0x00007fff5b2cad1a libdispatch.dylib`_dispatch_root_queue_drain + 325
    frame #14: 0x00007fff5b2cb4b1 libdispatch.dylib`_dispatch_worker_thread2 + 90
    frame #15: 0x00007fff5b4fc6ee libsystem_pthread.dylib`_pthread_wqthread + 619
    frame #16: 0x00007fff5b4fc415 libsystem_pthread.dylib`start_wqthread + 13

https://gsa.apple.com/grandslam/GsService2

{
    "Content-Type" = "text/x-xml-plist";
    "X-MMe-Client-Info" = "<MacBookPro14,3> <Mac OS X;10.14;18A391> <com.apple.AuthKit/1 (com.apple.akd/1.0)>";
}

# Send up machine info?

Request 3

(lldb) bt
* thread #4, queue = 'com.apple.authkit.url', stop reason = breakpoint 1.1
  * frame #0: 0x00007fff2d135bb5 CFNetwork`-[NSURLSession dataTaskWithRequest:completionHandler:]
    frame #1: 0x00007fff3f3d770f AuthKit`__59-[AKURLSession beginDataTaskWithRequest:completionHandler:]_block_invoke + 134
    frame #2: 0x00007fff5b2bbd4f libdispatch.dylib`_dispatch_call_block_and_release + 12
    frame #3: 0x00007fff5b2bcdcb libdispatch.dylib`_dispatch_client_callout + 8
    frame #4: 0x00007fff5b2c3120 libdispatch.dylib`_dispatch_lane_serial_drain + 618
    frame #5: 0x00007fff5b2c3bd8 libdispatch.dylib`_dispatch_lane_invoke + 388
    frame #6: 0x00007fff5b2cc084 libdispatch.dylib`_dispatch_workloop_worker_thread + 603
    frame #7: 0x00007fff5b4fc61c libsystem_pthread.dylib`_pthread_wqthread + 409
    frame #8: 0x00007fff5b4fc415 libsystem_pthread.dylib`start_wqthread + 13
(lldb) po $rdx
<NSMutableURLRequest: 0x7ff74dd7c020> { URL: https://gsa.apple.com/grandslam/MidService/syncMachine }

{
    "X-Apple-Client-App-Name" = "Apple Configurat";
    "X-Apple-I-Client-Time" = "2019-04-29T19:07:59Z";
    "X-Apple-I-MD-LU" = 74395C347FB810CAFBF0589D426A7DDAB71B19D75CFD69BC664EDD89ABC6F5DF;
    "X-Apple-I-MLB" = C018025031BHYRG1T;
    "X-Apple-I-ROM" = 4867f09f6ec5;
    "X-Apple-I-SRL-NO" = C02W34C3JUN9;
    "X-MMe-Client-Info" = "<MacBookPro14,3> <Mac OS X;10.14;18A391> <com.apple.AuthKit/1 (com.apple.akd/1.0)>";
    "X-Mme-Device-Id" = "164D1BC8-A71B-1F58-0C20-1BF98DB4F27D";
}

# Diff between GsService2 and postdata?

Request 7

(lldb) bt
* thread #26, queue = 'com.apple.authkit.url', stop reason = breakpoint 1.1
  * frame #0: 0x00007fff2d135bb5 CFNetwork`-[NSURLSession dataTaskWithRequest:completionHandler:]
    frame #1: 0x00007fff3f3d770f AuthKit`__59-[AKURLSession beginDataTaskWithRequest:completionHandler:]_block_invoke + 134
    frame #2: 0x00007fff5b2bbd4f libdispatch.dylib`_dispatch_call_block_and_release + 12
    frame #3: 0x00007fff5b2bcdcb libdispatch.dylib`_dispatch_client_callout + 8
    frame #4: 0x00007fff5b2c3120 libdispatch.dylib`_dispatch_lane_serial_drain + 618
    frame #5: 0x00007fff5b2c3bd8 libdispatch.dylib`_dispatch_lane_invoke + 388
    frame #6: 0x00007fff5b2cc084 libdispatch.dylib`_dispatch_workloop_worker_thread + 603
    frame #7: 0x00007fff5b4fc61c libsystem_pthread.dylib`_pthread_wqthread + 409
    frame #8: 0x00007fff5b4fc415 libsystem_pthread.dylib`start_wqthread + 13

<NSMutableURLRequest: 0x7ff74dd9eec0> { URL: https://gsas.apple.com/grandslam/GsService2/postdata }

{
    "X-Apple-HB-Token" = MDAxOTY1LTA1LTNiNWMwMGRiLWVkMDUtNDJiNS1iZjhkLTMzNGEzNjg3MDdhYjpBQUFBQkx3SUFBQUFBRnpIVVRrUkNtZHpMbWxrYlhNdWFHSzlBSjQwRjBJeDUzdUVtZWlReFZXNHErektETmxDY0VyaitpMXZiQzlFaUJXZ1d0MitvcDVwRjZuNnRROXZIVExSQ0lwV2h1dGVhMlhtMXJiT0NrMGQ3UndVR0F3TmhhQ2ZFSjRrNFppK1lwTkw3M2c2Ti9KaDlRSmVNRjNxS2R1Y1FBc2xNS0xpU1lWd05Id1czL2M2dU1KcS9nYzNQYmNwVjFPOFdCS2pXUkdYajZJVEk0c0RGNUFSUEowcWUrRGNOSER0OS9nVExPQ250S3JzQlFvcStyVjRsaHU5;
    "X-Apple-I-DeviceUserMode" = 0;
    "X-Apple-I-Locale" = "en_US";
    "X-Apple-I-MLB" = C018025031BHYRG1T;
    "X-Apple-I-ROM" = 4867f09f6ec5;
    "X-Apple-I-SRL-NO" = C02W34C3JUN9;
    "X-Apple-I-TimeZone" = "America/New_York";
    "X-Apple-I-TimeZone-Offset" = "-14400";
    "X-MMe-Client-Info" = "<MacBookPro14,3> <Mac OS X;10.14;18A391> <com.apple.AuthKit/1 (com.apple.akd/1.0)>";
    "X-Mme-Device-Id" = "164D1BC8-A71B-1F58-0C20-1BF98DB4F27D";
}

# Where to go from here?

CommerceKit

adamID = App Store product id

https://support.apple.com/en-us/HT203160

CommerceKit

DownloadOperationD
AssetDownloadOperation
HashedDownloadProvider
DecryptOperation
FairPlayHelper

ACUDownloadMobileAppOperation
CKStoreClient
CKStoreAccount


ISStoreClient
ISServiceProxu
CKStoreClient

https://github.com/mas-cli/mas/tree/master/MasKit/AppStore

ISServiceProxy

What does the real work?

storeaccountd
* Logging in (AccountServiceInterface)

storedownloadd
* Downloading things
* 

https://github.com/mas-cli/mas/issues/164

CommerceCore

ISPurchaseReceipt
asn1Token
asn1SetToken
asn1SequenceToken
asn1IntegerToken
asn1OSToken
asn1ReceiptToken

B64 encode and decode

Get MAC address

_parseISO8601

_sAppleROOTCert
_sWWDRCACert

Maybe the smallest and most foundational

StoreFoundation

ISServiceProxy main class
Provides access to 6 main XPC interfaces for interacting with the store.

performSynchronousBlock
objectProxyForServiceName

/System/Library/PrivateFrameworks/CommerceKit.framework/Versions/A/Resources/storeaccountd
serviceName: com.apple.storeaccountd.daemon
protocol: ISAccountService
interfaceClassName: AccountServiceInterface

/System/Library/PrivateFrameworks/CommerceKit.framework/Versions/A/Resources/storeuid.app/Contents/MacOS/storeuid
serviceName: com.apple.storeuid
protocol: ISUIService
interfaceClassName: UIServiceInterface

/System/Library/PrivateFrameworks/CommerceKit.framework/Versions/A/XPCServices/com.apple.CommerceKit.TransactionService.xpc
serviceName: com.apple.CommerceKit.TransactionService
protocol: ISTransactionService
interfaceClassName: TransactionServiceInterface

/System/Library/PrivateFrameworks/CommerceKit.framework/Versions/A/Resources/storeassetd<
serviceName: com.apple.storeassetd.daemon
protocol: ISAssetService
interfaceClassName: AssetServiceInterface

/System/Library/PrivateFrameworks/CommerceKit.framework/Versions/A/Resources/storedownloadd
serviceName: com.apple.storedownloadd.daemon
protocol: ISDownloadService
interfaceClassName: DownloadServiceInterface

/System/Library/PrivateFrameworks/CommerceKit.framework/Versions/A/Resources/storeuid.app/Contents/MacOS/storeuid
serviceName: com.apple.storeagent.storekit
protocol: ISInAppService
interfaceClassName: INAppServiceInterface


serviceName: com.apple.commerced
CommerceService

CKAccountStore
