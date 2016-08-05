+++
date = "2016-07-18T15:21:19-04:00"
draft = false
image = "firebase-controller/hero.png"
title = "KMHFirebaseController: A Cleaner API For Firebase"

description = "Firebase is an easy-to-use cross-platform NoSQL database, but even \"easy-to-use\" solutions can be \"tiresome-to-learn\" and messy when you're used to other tools. As such, I've created my own custom interface to make integrating Firebase into your iOS app even easier!"

+++

<div class="figure">
<img src="/blog/images/firebase-controller/firebase_apple.png">
</div>

[**Firebase**](https://firebase.google.com/) is an easy-to-use cross-platform [NoSQL](https://en.wikipedia.org/wiki/NoSQL) database, but even "easy-to-use" solutions can be "tiresome-to-learn" and messy when you're used to other tools. While Firebase's existing API is pretty robust, there are definitely some quirks to how it operates. As such, I've created my own custom interface to make integrating Firebase into your iOS app *even easier!*

This custom class&mdash;called, simply enough, **KMHFirebaseController**&mdash;is [available on GitHub](https://github.com/kenmhaggerty/KMHFirebaseController). In the past I've generally kept tools like these to myself (mostly out of lack-of-confidence), but in open sourcing this code I hope I can receive constructive criticism and recommendations for improvement. So fork away and submit your pull requests! :)

KMHFirebaseController contains class methods for setting up your Firebase project; pushing values to Firebase; querying Firebase; and observing Firebase. This library also includes a helper class called  **KMHFirebaseQuery** for use in creating Firebase queries.

The [KMHFirebaseController.h](https://github.com/kenmhaggerty/KMHFirebaseController/blob/master/KMHFirebaseController.h) file contains the full API, divided into the four general sections listed above and outlined below. Since my API is built on top of the existing Firebase iOS SDK, here is a link to the actual [Firebase iOS SDK documentation](https://firebase.google.com/docs/reference/ios/), which I would still recommend familiarizing yourself with.

This current version of KMHFirebaseController is compatible with Firebase 3.0 and later and written in Objective-C, but I'm hoping to push a backwards-compatible version soon in addition to an implementation in Swift.

And don't forget to add Firebase to your project by following the [**Get Started Guide**](https://firebase.google.com/docs/ios/setup)!

# KMHFirebaseController

## General Methods

#### + (void)setup
Performs necessary application setup for Firebase and KMHFirebaseController; should be called from within `-[UIApplicationDelegate application:didFinishLaunchingWithOptions:]`.

#### + (BOOL)isConnected
Returns whether there is currently an active connection to Firebase.

#### + (void)connect
Enables a connection to Firebase if it has been previously disabled.

#### + (void)disconnect
Disables a connection to Firebase if it is currently active.

## Data Methods

#### + (void)setPriority:(id)priority forPath:(NSString *)path withCompletion:(void(^)(BOOL success, NSError *error))completionBlock
As per [Firebase's documentation](https://firebase.google.com/docs/reference/ios/firebasedatabase/interface_f_i_r_database_reference#a137281e08179107ae2d575328facf825), priorities can be set per path to determine the order in which updates occur should sibling paths update simultaneously. Paths with no priority are processed first, followed by paths with numeric priorities and finally paths with string priorities.

#### + (void)saveObject:(id)object toPath:(NSString *)path withCompletion:(void (^)(BOOL success, NSError *error))completionBlock
Overwrites the object at the specified path with the given object and performs the given block upon completion. Valid object types include strings (NSString) and numbers and booleans (NSNumber). Setting the object for path to nil will remove that path from Firebase.

#### + (void)updateObjectAtPath:(NSString *)path withDictionary:(NSDictionary *)dictionary andCompletion:(void (^)(BOOL success, NSError *error))completionBlock
Sets the provided JSON object (NSDictionary) to the given path, overwriting existing objects at specified paths but leaving unspecified children objects untouched.

#### + (void)setOfflineValue:(id)offlineValue forObjectAtPath:(NSString *)path withPersistence:(BOOL)persist andCompletion:(void (^)(BOOL success, NSError *error))completionBlock
Firebase can automatically set an object to a path when the device's connection to Firebase goes offline. However, in the stock API this actions is performed only once. This method allows you to persist an offline value that will be set to the provided path each and every time your device disconnects from Firebase.

#### + (void)setOnlineValue:(id)onlineValue forObjectAtPath:(NSString *)path withPersistence:(BOOL)persist
Firebase can automatically set an object to a path when the device's connection to Firebase is established. However, in the stock API this actions is performed only once. This method allows you to persist an online value that will be set to the provided path each and every time your device connects to Firebase.

#### + (void)persistOnlineValueForObjectAtPath:(NSString *)path
This method, when called while an active Firebase connection is present, allows you to persist and resume the current value at the specified path whenever the device is connected to Firebase. If this method is not called while an offline value is set for the path, the prior value will not be re-set upon connection to Firebase.

#### + (void)clearOfflineValueForObjectAtPath:(NSString *)path
If an offline value has been set for the given path, this method will clear it.

#### + (void)clearOnlineValueForObjectAtPath:(NSString *)path
If an online value has been set for the given path, this method will clear it.

#### + (void)clearPersistedValueForObjectAtPath:(NSString *)path
If a persisted value has been set for the given path, this method will clear it.

## Query Methods

#### + (void)getObjectAtPath:(NSString *)path withCompletion:(void (^)(id object))completionBlock
Returns the object at the specified path via the completion handler.

#### + (void)getObjectsAtPath:(NSString *)path withQueries:(NSArray <KMHFirebaseQuery *> *)queries andCompletion:(void (^)(id result))completionBlock
Returns all child objects of the specified path matching the provided query constraints via the completion handler. Note that in order for this method not to error the authenticated user must have read access to all descendants of the specified path including children-of-children.

## Observer Methods
*N.b. For all observer methods, an object or child object must exist at the specified path in order to add an observer. In contrast to the stock API, methods below will call the provided responder blocks exactly once per update and will not call blocks when observers are initially added. Also in contrast with the stock API, the methods below allow for only one responder block per path per observation type (i.s., value changed, child added, child changed, and child removed).*

#### + (void)observeValueChangedAtPath:(NSString *)path withBlock:(void (^)(id value))block
Adds an observer at the specified path that will trigger the provided block when the object at the given path is changed.

#### + (void)observeChildAddedAtPath:(NSString *)path withBlock:(void (^)(id child))block
Adds an observer at the specified path that will trigger the provided block when any number of child objects are added at the given path.

#### + (void)observeChildChangedAtPath:(NSString *)path withBlock:(void (^)(id child))block
Adds an observer at the specified path that will trigger the provided block when any child object at the given path changes.

#### + (void)observeChildRemovedFromPath:(NSString *)path withBlock:(void (^)(id child))block
Adds an observer at the specified path that will trigger the provided block when any child object of the given path is removed.

#### + (void)removeValueChangedObserverAtPath:(NSString *)path
Removes the "value changed" observer from the given path.

#### + (void)removeChildAddedObserverAtPath:(NSString *)path
Removes the "child added" observer from the given path.

#### + (void)removeChildChangedObserverAtPath:(NSString *)path
Removes the "child changed" observer from the given path.

#### + (void)removeChildRemovedObserverAtPath:(NSString *)path
Removes the "child removed" observer from the given path.
