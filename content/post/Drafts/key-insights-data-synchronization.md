+++
date = "2016-06-30T11:09:52-04:00"
draft = true
title = "7 Key Insights In Data Synchronization"

+++

# NEEDS INTRO

## 1. Push Changes On Save via NSNotifications

I am a huge advocate of NSNotifications. They serve a similar utility as delegates but without the need to limit or declare who can respond. NSNotifications are also cleaner than key-value observing, as key-value observing requires exposing observed keys in your public interface. I use delegates when there is a clear 1-to-1 one-way dependency (e.g., from an interface to its data source) and instead use NSNotifications wherever and whenever key actions take place that may trigger behaviors in one-to-many functionally independent classes (e.g., updating the interface when the user updates their profile photo).

(And no, I don't use key-value observing unless there is truly no other way.)

Pretty much every app I build uses NSNotifications in this way, so I'm used to implementing custom setters within my data classes to broadcast value changes, e.g.:

<script src="https://gist.github.com/kenmhaggerty/eb013fde749156d8780f449715f43079.js"></script>

Two things to notice in the above code snippet:

- I check whether the old and new values are equal and return if they are. This avoids broadcasting change notifications when the value hasn't actually changed.
- I set the value of `name` to `@"newValue"` in `userInfo` only if `name` is not nil. (You can't set a key in NSDictionary to nil.)

When it comes to data synchronization, however, we're less interested in pushing changes to our remote database as soon they're made and more interested in pushing them as soon as they're persisted. This prevents us from attempting to push incomplete states to our server, especially if our backend performs server-side validation on our data.

Thankfully, if we're using Core Data to persist our data locally, we can write similar notifications to broadcast when our local changes are saved using the `willSave` and `didSave` methods in NSManagedObject:

<script src="https://gist.github.com/kenmhaggerty/cbdd5f38b329eab2384166730924781b.js"></script>

Notice that I've also added an NSSet `changedKeys` to hold strings of all the properties whose values were changed when the object itself was saved. I find that it’s useful to send out an NSNotification for each specific attribute where the value has changed in addition to when the NSManagedObject itself is saved. Doing so can help streamline our synchronization process by pushing only those values that have changed rather than every property on every object that has changes.




When I first started out programming I decided I wanted to learn three key things:

- How to build a user interface
- How to persist data
- How to synchronize data

These, to me, represented fundamentally what it means to build an app.

* * *

In this post I'll discuss how I approached building an app for data synchronization.



 NSNotifications are useful for broadcasting two key situations: when an attribute's value is changed or when the NSManagedObject itself is saved. Each of these situations satisfies a specific application need: our didChange notifications trigger us to update relevant portions of our user interface, while didSave notifications trigger us to persist our local changes to our remote database.

Triggering a didChange notification is easy enough. I simply write a custom setter for each attribute where I want an NSNotification to be sent out, e.g.:

<script src="https://gist.github.com/kenmhaggerty/fe92195b27bde93d5b0f123eaf22d0c3.js"></script>

Note that our setter syntax for NSManagedObject attributes is a bit more verbose since NSManagedObject attributes are dynamic rather than synthesized. However, this code behaves just like setting your instance variable in a synthesized property.

To trigger a didSave notification, I simply overwrite the didSave method in my NSManagedObject subclass:

<script src="https://gist.github.com/kenmhaggerty/b03a19191e7b71f401c0c532324bf743.js"></script>

However, I find that it’s useful not only to send out a didSave notification for the NSManagedObject as a whole but also for each specific attribute where the value has changed. Doing so can help streamline our synchronization process by only updating those values that have changed rather than every property on every object that has changed. NSManagedObject conveniently has a property called changedValues which is an NSDictionary of every attribute whose value was changed and the new value of each key; however, by the time didSave is called, changedValues is set back to nil. To compensate, we can create our own property on NSManagedObject called changedKeys (an NSSet of NSStrings), set it during willSave, and utilize it during didSave as follows:

<script src="https://gist.github.com/kenmhaggerty/cbdd5f38b329eab2384166730924781b.js"></script>

## 2. isDownloaded

It’s incredibly useful to distinguish whether a newly created NSManagedObject was created locally or based on information fetched from our remote database. Failure to do so can result in an infinite loop, as a newly downloaded object can be mistaken for a newly created object that is then pushed to our remote database and re-downloaded ad infinitum. To compensate, I like to create a custom property isDownloaded whose value is NO by default and YES whenever the object is changed because of changes to our corresponding object in our remote database. I then set isDownloaded back to NO after those changes are saved locally, making sure in the process not to push these changes back to the remote database as this may cause your backend to broadcast retrigger a didChange notification.

## 3. isUploaded

Similarly, it’s also useful to keep track of whether a local object is missing its corresponding object in our remote database because it is newly created or because our remote object was deleted (and should therefore cause us to delete our local object). Depending on the granularity at which you want to track where all your local instances of your remote object exist, you can either create a server-side property called localInstances to track each and every local instance of a remote object, or you can create a persistent client-side property on each NSManagedObject called isUploaded to track whether that object has been successfully pushed to our remote database at least once. Choosing whether to track individual instances is really dependent on the complexity and specific needs of your app, as I’ve made apps both where this is essential and where this is overkill.

## 4. createdAt/instantiatedAt/updatedAt vs. creationDate/editDate + lastSyncDate

NSManagedObject by default has no properties that track when the NSManagedObject was first created, instantiated, or last edited. Adding these properties is easy enough. The trick is to remember that programmatic creation and edit dates are not the same as user-facing creation and edit dates. For example, an NSManagedObject created by fetching values from our remote database will have a later local creation date than what should be displayed to our user. Additionally, since not every syncable change to our NSManagedObject will be the result of user action, the edit date displayed to our user will likewise vary from when our NSManagedObject was last updated. As a result, I like to add the following four attributes to my NSManagedObjects:

- createdAt: when an NSManagedObject is first added to our local database
- updatedAt: when an NSManagedObject was last changed in any capacity
- creationDate: when an NSManagedObject was first created by a user, independent of when it was actually first created in Core Data
- editDate: when an NSManagedObject was last edited by a user, independent of when it was actually last updated in Core Data

Additionally, you can also create a custom property instantiatedAt that is set when the NSManagedObject is initialized or fetched. I myself haven’t yet found a compelling use case for instantiatedAt, but it is certainly a property I would otherwise have assumed was built into NSManagedObject.

## 5. Single purpose properties

One of my key realizations during data synchronization was that an ordered list of items is really two properties in one and should be treated as such to reduce overhead. Instead of saving ordered lists as NSOrderedSets in Core Data, I instead save my ordered lists as an unordered NSSet of items and an ordered NSOrderedSet of corresponding IDs. That way, simply reordering items in a list is treated as such and doesn’t require either overwriting the entire list of objects or performing heavy introspection as to whether an updated list of objects contains the same elements as before and, if not, which were added or removed. This greatly streamlines our synchronization process.

## 6. isDeleted, wasDeleted, parentWasDeleted

As has been noted in other blogs, NSManagedObject’s isDeleted property is poorly named. Rather than signifying that an object has been deleted, isDeleted simply indicates that an NSManagedObject is irrevocably marked for deletion. Additionally, NSManagedObjects aren’t truly deleted from the NSManagedObjectContext until their didSave methods have finished executing. What this means is that deletions are not persisted until didSave is called, at which point the value of isDeleted is actually NO.

In order to trigger certain actions once the deletion of an NSManagedObject is truly persisted to Core Data, we can yet again create our own custom property wasDeleted whose value is set during willSave and whose value is not unset automatically by didSave. Then, during didSave, we can check the value of wasDeleted to see if our newly saved object was indeed deleted from our database and, if so, to take certain actions as a result (e.g., to delete our corresponding data object from our remote database).

In addition, it is also useful to note whether a deleted object was deleted because it itself was deleted or because its parent object was deleted. (E.g., whether a question itself was deleted, or whether the survey to which the question belonged was deleted.) To discern this, we yet again create our own custom property parentWasDeleted whose default value is NO. When a parent object is deleted, we set each child object’s parentWasDeleted property to YES during prepareForDeletion. Additionally, in the setter for parentWasDeleted, we can propagate our value to each object’s child objects. This helps us ensure that we understand during synchronization how to handle deletions in the case that the order in which we process objects cannot be well regulated.

## 7. Deletion markers

Because our remote database serves as our canonical and universally accessible representation of our data model, it’s important that key information not be removed until it is fully utilized. For example, if an object is deleted from our data model, we need to persist both some indicator within our remote database that our object has been deleted in addition to whatever key values are necessary to pinpoint our specific object. Depending on the architecture of our app, this may or may not require maintaining a list in our remote database of every local instance of our remote object. However, it will always require a two-step deletion process: first, a universally accessible notification that our object ought to be deleted, followed by actually deleting the object from our remote database.
