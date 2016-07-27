+++
date = "2016-07-26T15:57:58-04:00"
draft = false
title = "Shared Objects In Firebase, Part 1: Security Rules"

description = "I had originally planned to publish a blog post on I couldn't figure out how to implement sharing in Firebase. However, in the process of writing my blog post, I solved my own problem! Read on to see how."

+++

### Deus Ex Machina: or, in Praise Of Writing

> "I write entirely to find out what I’m thinking, what I’m looking at, what I see and what it means. What I want and what I fear."  
> &mdash; Joan Didion, ["Why I Write"](http://www.montgomeryschoolsmd.org/uploadedFiles/schools/whitmanhs/academics/english/Why%20I%20Write%20Didion.pdf)

I had originally planned to publish a blog post on my difficulties working with Firebase. I had managed to create a cleaner API for Firebase ([**KMHFirebaseController**](/blog/post/firebase-controller)), but I struggled to find a way to share objects in Firebase similar to how I did in [Parse](http://www.parse.com/)&mdash;i.e., user- and object-specific; semantically naïve; as [normalized](https://en.wikipedia.org/wiki/Database_normalization) as possible; and with correct read, write, and sharing permissions.

I spent the two weeks after my second blog post attempting to implement this before deciding it was a better use of time to publish what I had and implement sharing later.

I spent a few more days making one last attempt before putting the problem on the back burner. Instead, I wrote about my goals and challenges implementing sharing in Firebase, hoping that it might attract answers better than a Stack Overflow post and that I might have it done before a job interview.

As it happens, it was the process of verbalizing my predicament which gave words to my solution. I solved my own problem! Classic [rubber duck debugging](https://en.wikipedia.org/wiki/Rubber_duck_debugging), but without the rubber duck.

### So What Was The `root` Of The Problem?

First, a little background on Firebase.

Firebase is a [NoSQL](https://www.mongodb.com/nosql-explained) database. That means that, unlike a relational database like SQL, arbitrary pieces of data can't be tied together using pointers. NoSQL databases come in many varieties; in Firebase, data is highly nested. Firebase uses a JSON-like tree structure where each node must hold either a boolean, a string, a value, or another node. This makes normalizing our database both more difficult and less of a priority than in a relational database.

Additionally, permissions in Firebase are determined by static, server-side security rules. This is in contrast to Parse, where roles and access control lists can be created and set client-side.

Thirdly, security rules in Firebase cascade, wherein the rules set by parent nodes overrule those set by their children.

Finally, Firebase restricts queries to nodes where the requesting user has access to every descendant of the node. Attempting to query on a node where some objects cannot be read results in an error. This is also in contrast to Parse, where read permissions also act as filters on queries.

My first and most Parse-like attempt was to attach custom permission values to each data object:

<script src="https://gist.github.com/kenmhaggerty/ae366672face971ae02606d5bd0be9b8.js"></script>

While this created a very normalized schema, it didn't allow us to perform client-side observation of added or removed permissions, as Firebase doesn't allow us to observe objects we don't have access to. It also doesn't allow us to figure out which objects each user has access to, as there's no way to query, filter, or fetch accessible object IDs.

What we needed is a schema where a user can query a single node that only they can access in order to fetch a list of IDs for all the objects they have access to. We can do this by extracting out our permissions objects into a new `permissions` node separate from our `objects` node:

<script src="https://gist.github.com/kenmhaggerty/3959745efb6ac8b2cb1a16d7332f44ac.js"></script>

This schema provides us with two clear endpoints for our user to observe: `permissions/public` for publicly-shared objects and `permissions/user/$user_id` where `$user_id` is a UUID for the current user. However, in this schema, it's not clear what the security rules should be for the permissions objects themselves. For example, who can decide whether an object should be public?

The revelation I had, as hinted at above, was that security rules need not only depend on the data at the current node but *anywhere* in the database by referring to `root`. For example, a simple security rule might allow read and write access only if the current user's ID matches the current node and ensure that only strings can be set. Such a rule would be written as follows:

<script src="https://gist.github.com/kenmhaggerty/1c6395d2478cd12c6667086768aea918.js"></script>

However, if I wanted my security rule to refer to a piece of data elsewhere in my database, I could access that value using `root.child('path/to/value')`. This allows us to normalize our database into the following schema while preserving the observation and query endpoints we created by separating our permissions into a separate node:

<script src="https://gist.github.com/kenmhaggerty/e288bbcb0fe14312d32f6573354f1cf5.js"></script>

In this design, our user can still observe when permissions are added or removed by observing `permissions/public` and `permissions/user/$user_id`, but by using the following security rules, we ensure full data security:

<script src="https://gist.github.com/kenmhaggerty/2bb03d6c2e2ae9c6fbb9e4178810cbf2.js"></script>

These security rules are a little complex, but the high level overview is:

- All authenticated users can obtain a list of public objects.
- Only the appropriate user can see which objects have been specifically shared with that user.
- Only users with whom the object has been specifically shared can edit whether an object is public.
- Only users with whom the object has been specifically shared can share the object with others.
- Permissions can only be created for object IDs that exist.
- Users can always removes themselves from a shared object.
- New objects can only be created by authenticated users.

I've tested out these security rules in Firebase's invaluable new [Security Simulator](https://firebase.googleblog.com/2012/12/the-new-firebase-security-api.html), and from what I can see all of these requirements seem to be met. I'll have to write some tests to ensure that this is the case both in its current incarnation and in all future updates; but for now, I am definitely very proud.