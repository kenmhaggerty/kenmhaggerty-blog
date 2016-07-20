+++
date = "2016-07-04T17:58:44-04:00"
draft = false
image = "animating-tableview-updates/hero.png"
title = "A Better Way To Animate Table View Updates"

description = "Those of you familiar with UITableView methods are probably comfortable inserting, reordering, and deleting rows from your table view, but what happens if you want to perform a mix of these actions within a single animation?"

+++

Whenever I find myself re-using a helper method in many of my classes or projects, I like to add it to a shared class I created called [**KMHGenerics**](https://github.com/kenmhaggerty/KMHGenerics). This class contains generically useful methods so that my actual code can remain as ["dry"](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) as possible.

Today I'm going to talk about a method I wrote to animate complex updates to UITableViews. The KMHGenerics class shared [here on GitHub](https://github.com/kenmhaggerty/KMHGenerics) includes many other methods too, but I'll save those for another blog post.

Those of you familiar with UITableView methods are probably comfortable inserting, reordering, and deleting rows from your table view, but what happens if you want to perform a mix of these actions within a single animation? For example, how would you animate inserting a row at the end of your table view while simultaneously deleting the first row of your table view and reordering the remaing cells?

If you are efficient and looking for the simplest solution, the answer is to call `[self.tableview reloadData]` and update the entire table view at once. While this works, the lack of animations can be, in certain situations, extremely confusing to the user&mdash;much like the following GIF:

<div class="figure">
  <img src="/blog/images/animating-tableview-updates/change-blindness.gif">
</div>
<span class="caption">**Above** An example of [change blindness](https://en.wikipedia.org/wiki/Change_blindness). Can you spot the difference between the two images? A blank frame added between the two images breaks the visual continuity, making it much harder to spot changes. Similarly, when updates to UITableView are not animated, it can be very difficult to spot what changed. [[Source]](https://www.csc.ncsu.edu/faculty/healey/PP/)</span>

Instead, I've written the following method on UITableView to perform updates in a single animation:

<script src="https://gist.github.com/kenmhaggerty/9c0124d7684413fec140263a3c435aa9.js"></script>

This method takes in the follow ten parameters:

- **array**: The current state of the data supplying the table view.
- **toArray**: The desired final state of your table view's data.
- **section**: Which section of your table view to animate.<sup id="ref_1">[1](#footnote_1)</sup>
- **insertionAnimation**: What animation style to use when inserting new cells in your table view.
- **deletionAnimation**: What animation style to use when deleting cells from your table view.
- **setter**: A block that sets the data for your table view.
- **insertedCells**: An optional block to execute on the cells that will be inserted.
- **reorderedCells**: An optional block to execute on the cells that will be reodered.
- **deletedCells**: An optional block to execute on the cells that will be deleted.
- **completion**: An optional block to execute upon completion of all animations.

To help illustrate, I've embedded an interactive demo below showing. Try it out and let me know what you think!

<div class="figure">
  <iframe class="figureContent" src="https://appetize.io/embed/qkm9627b77pvrr7rqf831ty84r?device=iphone4s&scale=100&autoplay=true&orientation=portrait&screenOnly=true" width="320px" height="480px" scrolling="no"></iframe>
</div>
<span class="caption">**Above** Tap "Shuffle" to generate random data for the table view. By toggling the switch at top-left, you can see the difference between animated and instantaneous table view updates for yourself. [[Created using Appetize.io]](https://appetize.io)</span>

* * *
<p class="footnotes">
  <a name="footnote_1" href="#ref_1">1</a> Currently this method only supports updating one table view section at a time. At some point I should implement a version that supports updating multiple table view sections simultaneously, e.g., for when table view cells are moved between sections.
</p>
