+++
date = "2016-07-01T11:10:04-04:00"
draft = false
title = "App Architecture (or, Making Sure The Chef Isn't Performing Heart Surgery)"

description = "When it came to building an app that both persists data locally and synchronizes data with a remote server, I spent some time making sure that what I built not only worked but also made sense."

+++

When I think about classes in Objective-C I imagine running a restaurant. A well run restaurant not only prepares good food but also provides excellent service and manages employees well. A good restaurant also delegates out whatever it cannot or should not do itself. For example: if a guest at your restaurant were in need of medical attention, you would probably call an ambulance and not ask your chef to attempt heart surgery (no matter how good he or she may be with knives).

A well-built app follows similar principles. Your core functionalities should be grouped and split between classes so that each class has one area of expertise. Classes should communicate and depend on each other in a clear and coordinated fashion and not haphazardly. Classes should be independent of each other and self-contained. And certain things that are too complicated for you to do yourself or that are not core to the app (e.g., your backend) should be outsourced to third party libraries, frameworks, services, and APIs.

When it came to building an app that both persists data locally and synchronizes data with a remote server (in this case, [Firebase](https://www.firebase.com/)), I spent some time making sure that what I built not only _worked_ but also _made sense_. This involved a lot of trial and error, building and rebuilding, and, of course, refactoring. Ultimately, though, I've come around to the following architecture which, to me, seems pretty universal:

<div class="figure">
<img src="https://docs.google.com/drawings/d/1YN09kkYLQymJxVmq-IKgZCWBX5KSdaOVN6uZKN-g5xk/pub?w=487&amp;h=383">
</div>
<span class="caption">**Above** Abstracting apart classes for managing local data storage (Core Data Controller), remote data storage (Firebase Controller), and synchronization (Sync Engine) allows for a cleaner interface with our app (Data API) and greater flexibility should we need to change backend providers.</span>

In the above diagram, all requests for data are routed through the **Data Manager**. This includes everything from fetching data to deleting data to synchronizing data. Requests involving local data are then routed to our **Core Data Controller**, while requests involving our remote database are routed to our **Sync Engine**. Our Core Data Controller then interacts with our data objects directly (in this case, our **Managed Objects**), while our Sync Engine directs remote database requests to Firebase via our intermediary **Firebase Controller** class.

The arrows in the diagram indicate the logical flow between classes. They also represent dependencies between classes. To avoid circular dependencies, there should be no double-headed arrows; the only exception here is between our Sync Engine and our backend (i.e., Firebase Controller).<sup id="ref_1">[1](#footnote_1)</sup>

Some of the arrows between our classes have dotted lines. That's just to show that the connection between these classes are weaker. The dotted line from our Data Manager to our Sync Engine simply indicates that these two classes are connected only so that our Data Manager can manually trigger a sync if necessary (e.g., when the user taps a reload button). Meanwhile, the dotted lines between our Managed Objects and our Sync Engine represent NSNotifications sent out by our Managed Objects when certain key values change; our Sync Engine listens for these notifications and, in response, pushes these changes to our remote server via our Firebase Controller.

Our diagram above is also organized so that our most app-facing classes are at the top and our most back-end facing classes are towards the bottom.

This diagram also includes a **Notifications Controller** for scheduling and managing local notifications. When our app receives a local notification, our **App Delegate** handles the response. While local notifications aren't part of our synchonization scheme, you can see how its architecture resembles that of our Firebase Controller. This allows our app to be easily extended without extensive refactoring.

If organizing the "model" component of our [MVC architecture](https://developer.apple.com/library/ios/documentation/General/Conceptual/DevPedia-CocoaCore/MVC.html) in this way makes sense, then we can begin to investigate how actually to implement it in an app. In my next few blog posts, I'll delve deeper into how I did this in my newest app [PushQuery](https://www.appstore.com/pushquery), including <a>tips and tricks to working with Core Data</a>; <a>using protocols to abstract implementation from interface</a>; <a>realtime synchronization via Firebase</a>; and <a>how to schedule and update local notifications</a>.

Questions? Feedback? Let me know!  
[kenmhaggerty@gmail.com](mailto:kenmhaggerty@gmail.com?Subject=Re:%20[Blog]%20App%20Architecture)

* * *
<p class="footnotes">
<a name="footnote_1" href="#ref_1">1</a> Our Sync Engine is a delegate of our Firebase Controller that updates our local data store when our Firebase Controller receives updates from our remote server. This architecture is specific but not unique to the way Firebase works.
</p>
