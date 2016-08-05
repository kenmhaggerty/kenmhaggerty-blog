+++
date = "2016-08-04T17:48:00-04:00"
draft = false
title = "Shared Objects In Firebase, Part 2: KMHFirebaseController+ACL"

description = "In Part 1, I outlined how I set security rules and architected my data to allow for ACL-like object sharing in Firebase. In this post, I’ll go over KMHFirebaseController+ACL, a category built on top of KMHFirebaseController that allows you to set user-based object permissions via a single method call, and I'll explain how it works and how to use it in your app."

+++

In [Part 1](/blog/post/firebase-acl-part-1/), I outlined how I set security rules and architected my data to allow for [ACL](https://en.wikipedia.org/wiki/Access_control_list)-like object sharing in [Firebase](https://firebase.google.com/). However, I never went over how I actually implemented shared objects in Firebase. In this post, I'll go over [**KMHFirebaseController+ACL**](https://github.com/kenmhaggerty/KMHFirebaseController), a category built on top of [KMHFirebaseController](/blog/post/firebase-controller/), and explain how it works and how to use it in your app.

### How It Works

As I described in my previous post, KMHFirebaseController+ACL works by using a set of server-side [security rules](https://gist.github.com/kenmhaggerty/2bb03d6c2e2ae9c6fbb9e4178810cbf2) to limit who can read and write to objects on Firebase. That itself seems simple enough, but the fact that security rules can't be set or edited via the client (i.e., your app) means that we need to be creative with how we can use Firebase objects themselves as a means of setting access controls through our app without having to edit our static security rules.

To do this without compromising data security, we establish the following procedure for saving objects to Firebase.

1. First, ensure that:
  - The user is authenticated (i.e., logged in).
  - Our data object is formatted correctly for Firebase.
  - A unique ID has been assigned to our data object.
2. Grant ourself permission to create an object on Firebase with our desired object ID.
3. Save our object to Firebase.
4. Set permissions for our object on Firebase, including whether our object should be publicly viewable by all authenticated users of your app and which other users should have specific access to read, edit, and modify permissions for our object.

Attempting to save or fetch objects without following this procedure will result in an error.

Because we like to keep our code ["dry,"](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) this procedure has been cleanly encapsulated within KMHFirebaseController+ACL. All you have to do is provide the necessary parameters and callbacks for each method call.

### Key Assumptions

Because certain emergent behaviors are hard-coded into the data architecture and security rules, there are a few key assumptions that must be made when using KMHFirebaseController+ACL:

- Objects on Firebase are only accessible to logged-in users.
- Every object you store on Firebase has a unique ID.
- Child objects cannot have different permissions than their parents. (Instead, child objects should be stored as sibling objects and referenced and fetched via their object ID.)
- Public objects can be read by any logged-in user but can only edited by users with whom the object has been explicitly shared ("specified users").
- Object sharing permissions can only be edited by specified users.
- There is currently only one level of permission granted to objects (i.e., "specified user").
- There is currently no way to perform semantic queries over accessible objects on Firebase. (Objects must be fetched individually and queried locally.)

Future versions of KMHFirebaseController+ACL will hopefully require fewer assumptions to allow for greater flexibility. For example, I myself would like to add the ability to create custom roles to add more granular permissions to objects on Firebase (e.g., specifying whether a user should be able to delete an object).

### Thoughts? Feedback? Found a bug?

If you have any ideas for cool features or improvements, please fork, edit, and pull request away! All of this code is [available on GitHub](https://github.com/kenmhaggerty/KMHFirebaseController) as part of KMHFirebaseController. Or, you can always email me at [kenmhaggerty@gmail.com](mailto:kenmhaggerty@gmail.com?Subject=Re:%20[Blog]%20App%20Architecture) or message me on Twitter at [@kenmhaggerty](https://twitter.com/kenmhaggerty).

And without further ado, here's the full API!

# KMHFirebaseController+ACL

## Notifications
*The following constants are notification names which you can observe via the default [NSNotificationCenter](https://developer.apple.com/reference/foundation/nsnotificationcenter?language=objc). To get an NSSet of NSStrings for object IDs, access the value of* `FirebaseNotificationUserInfoKey` *within the notification's* `userInfo` *dictionary.*

#### FirebaseObjectsWithIdsWereMadePublicNotification
Includes IDs for both newly accessible and previously accessible objects that were made public.

#### FirebaseObjectsWithIdsWereMadePrivateNotification
Includes IDs for only those objects to which the current user still has access.

#### FirebaseObjectsWithIdsWereSharedNotification
Includes IDs for both newly accessible and previously accessible objects that were shared with the current user.

#### FirebaseObjectsWithIdsWereUnsharedNotification
Includes IDs for only those objects to which the current user still has access.

#### FirebaseObjectsWithIdsWereMadeInaccessibleNotification
Includes IDs for objects to which the current user no longer has access, whether because the item was unshared or because the item was made private and not shared with the current user.

## Setup

#### + (void)setCurrentUserId:(NSString *)userId
Needed in order observe when objects are shared or unshared and to save new objects to Firebase. The provided `userId` should be identical to `uid` in [FIRUserInfo](https://firebase.google.com/docs/reference/ios/firebaseauth/protocol_f_i_r_user_info-p) obtained via Firebase authentication, as it is used to determine access to data. Simultaneously, you should ensure that the user is indeed authenticated. In the schema used by KMHFirebaseController+ACL, unauthenticated users do not have access to data on Firebase, so `userId` should be `nil` whenever the user is logged out.

## Save

#### + (void)saveObject:(id)object withId:(NSString *)objectId isPublic:(BOOL)isPublic users:(NSSet <NSString *> *)userIds error:(void (^)(NSError *error))errorBlock completion:(void (^)(BOOL success))completionBlock
For saving new objects to Firebase. The current user is automatically included as a specified user and does not need to be included in `userIds`. Because this method performs multiple concurrent calls to Firebase, `errorBlock` may be called multiple times.

#### + (void)overwriteObjectWithId:(NSString *)objectId withObject:(id)object andCompletion:(void (^)(BOOL success, NSError *error))completionBlock
For updating objects on Firebase. An object with the provided objectId must already exist on Firebase, and the current user must already have specific access to it.

## Permissions

#### + (void)setObjectWithId:(NSString *)objectId asPublic:(BOOL)isPublic withCompletion:(void (^)(BOOL success, NSError *error))completionBlock
Public objects can be read by all users, while private object objects can only be read by specified users. Only specified users can edit objects. The current user must be a specified user in order to change object privacy settings.

#### + (void)addUserWithId:(NSString *)userId toObjectWithId:(NSString *)objectId withCompletion:(void (^)(BOOL success, NSError *error))completionBlock
Allows the specified user to read and edit the object on Firebase regardless of whether the object is public or private. The current user must be a specified user in order to add another user to the object.

#### + (void)removeUserWithId:(NSString *)userId fromObjectWithId:(NSString *)objectId withCompletion:(void (^)(BOOL success, NSError *error))completionBlock
Removes the specified user from the object. The user will no longer be able to edit the object but will be able to continue reading the object if it is public. The current user must be a specified user in order to remove a user from the object.

## Fetch

#### + (void)objectExistsWithId:(NSString *)objectId error:(void (^)(NSError *error))errorBlock success:(void (^)(BOOL exists))successBlock
Checks to see if an object with the specified `objectId` already exists on Firebase.

#### + (void)fetchObjectWithId:(NSString *)objectId completion:(void (^)(id object))completionBlock
Fetches and returns the raw Firebase object via the provided `completionBlock`. The current user must have read access to the object in order to fetch its contents.

#### + (void)fetchPublicObjectsWithBlock:(void (^)(id object, float progress))block
Fetches and iteratively returns all public objects. For each object, the provided `block` is called once and also includes a `progress` value ranging from `0.0` to `1.0`, where each object is counted equally towards progress. Note that objects may be both public and shared.

#### + (void)fetchSharedObjectsWithBlock:(void (^)(id object, float progress))block
Fetches and iteratively returns all objects shared with the current user. For each object, the provided `block` is called once and also includes a `progress` value ranging from `0.0` to `1.0`, where each object is counted equally towards progress. Note that objects may be both shared and public.

## Observe
*These observer methods follow the same behavior established in [KMHFirebaseController](/blog/post/firebase-controller/).*

#### + (void)observeValueChangedAtPath:(NSString *)path forObjectWithId:(NSString *)objectId withBlock:(void(^)(id value))block
Adds an observer at the given path relative to the provided `objectId` that will trigger the provided block when the value at that path is changed.

#### + (void)observeChildAddedAtPath:(NSString *)path forObjectWithId:(NSString *)objectId withBlock:(void(^)(id value))block
Adds an observer at the given path relative to the provided `objectId` that will trigger the provided block when any number of child objects are added at that path.

#### + (void)observeChildChangedAtPath:(NSString *)path forObjectWithId:(NSString *)objectId withBlock:(void(^)(id value))block
Adds an observer at the given path relative to the provided `objectId` that will trigger the provided block when any child object at the given path changes.

#### + (void)observeChildRemovedFromPath:(NSString *)path forObjectWithId:(NSString *)objectId withBlock:(void(^)(id value))block
Adds an observer at the given path relative to the provided `objectId` that will trigger the provided block when any child object at the given path is removed.

#### + (void)removeValueChangedObserverAtPath:(NSString *)path forObjectWithId:(NSString *)objectId
Removes the "value changed" observer from the given path relative to the provided `objectId`.

#### + (void)removeChildAddedObserverAtPath:(NSString *)path forObjectWithId:(NSString *)objectId
Removes the "child added" observer from the given path relative to the provided `objectId`.

#### + (void)removeChildChangedObserverAtPath:(NSString *)path forObjectWithId:(NSString *)objectId
Removes the "child changed" observer from the given path relative to the provided `objectId`.

#### + (void)removeChildRemovedObserverAtPath:(NSString *)path forObjectWithId:(NSString *)objectId
Removes the "child removed" observer from the given path relative to the provided `objectId`.
