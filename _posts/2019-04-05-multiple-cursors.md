---
layout: post
title: "Multiple Cursors in Visual Studio Code"
date: 2019-04-05
---

Programming using multiple cursors had always been a mystery to me. I had seen it before when digging through editor settings, but I had never bothered to actually try it until recently. Now I use it every day, and I keep finding new ways to speed up otherwise tedious tasks.

<!--more-->

Multiple cursors is exactly what it sounds like: it lets you place more than one cursor in your editor so you can edit multiple locations at the same time. It sounds like more trouble than it’s worth, but once you learn the shortcuts, it becomes natural.

Visual Studio Code is my go-to editor nowadays, and it comes with multiple cursor support out of the box. Here are the important shortcuts in Visual Studio Code (on Windows):

- `Alt+Click`: Place a new cursor where you click
- `Ctrl+Alt+Down`: Add a new cursor below your current one
- `Ctrl+Alt+Up`: Add a new cursor above your current one

That’s about it. Once you have your cursors in place, you can type as usual and your changes will apply to each cursor location.

{% include image.html file="2019-04-05-multiple-cursors/demo.gif" alt="GIF that shows typing on multiple lines using multiple cursors" %}

Of course, your lines won’t always align so nicely, so you may have to use a clever combination of other navigation shortcuts to achieve your goal. Here are a few I use all the time:

- `Home`: Move cursor to beginning of line
- `End`: Move cursor to end of line
- `Ctrl+Left`: Move cursor one word left
- `Ctrl+Right`: Move cursor one word right
- `Shift+[any of above]`: Select from previous position to new position
- `Ctrl+Backspace`: Delete one word left
- `Ctrl+Delete`: Delete one word right

To illustrate, here are a few cases I’ve come across where I’ve used multiple cursors extensively.

# Condensing lines into one

Let’s start simple. Say you copied a list of names from somewhere:

```
Trout
Judge
Votto
Blackmon
Stanton
```
(Bonus points if you know why I chose those names.) And you wanted to put them into a comma-separated string. You could manually bring them into one line and add commas between. Or...

{% include image.html file="2019-04-05-multiple-cursors/condensing-lines.gif" alt="GIF that shows multiple lines being condensed into one line" %}

Wait… what just happened? Step-by-step:

1. Paste names (`Ctrl+V`)
1. Move to beginning of last line (`Ctrl+Left`)
1. Insert cursors on lines 4, 3, and 2 (`Ctrl+Alt+Up`, 3 times)
1. Bring all names into one line (`Backspace`)
1. Separate by commas

That’s just 8 separate keystrokes. You can imagine how much this would speed things up if you had a longer list.

# Appending to offset locations

Look back to the example in the intro. It was easy to place cursors using `Ctrl+Alt+Up` because we needed cursors at the same position on each line (i.e. they lined up vertically). What if we need them offset? We could use `Alt+Click` to place each cursor individually, but that would take a while if you had lots of lines. Instead, let’s take advantage of other navigation shortcuts.

{% include image.html file="2019-04-05-multiple-cursors/offset-append.gif" alt="GIF that shows multiple cursors being moved to vertically-offset positions" %}

Step-by-step:

1. Insert cursors on lines below (`Ctrl+Alt+Down`, 4 times)
1. Skip forward one word (`Ctrl+Right`)
1. Move to right of variable name (`Right`, 2 times)
1. Enter some text

Notice how after step 2, the cursors are vertically offset, but they’re all in the same position relative to the variables.

# Placing cursors using “Find”

The lines we’re interested in won’t always be on top of one another. They could be in different parts of the file. Fortunately, we can use good old “Find” to select all the areas we’re interested in and drop cursors on them.

For example, consider a file with several interfaces and classes. How can we select and copy the names of every interface and class?

{% include image.html file="2019-04-05-multiple-cursors/find.gif" alt="GIF that shows searching in a file and placing cursors at matched locations" %}

Step-by-step:

- Find all occurrences of “export” (`Ctrl+F`, type “export”, Esc)
- Select all of the find matches (`Ctrl+Shift+L`)
- Move cursors after interface/class names (`Ctrl+Right`, 2 times)
- Select interface/class names (`Ctrl+Shift+Left`)
- Copy (`Ctrl+C`)

I made this easier by using “export” as an anchor, but you can do more complex finds by using regular expressions. For example, if the “export” words weren’t there, then I could use the following regex to find every occurrence of "interface" or "class" that starts a line:

```
^(interface|class)
```

The rest of the process would be the same.

# Replacing multiple words with other words

Finally, let’s see how you can replace multiple words with different words simultaneously. Again, say you copied a list of 5 names from somewhere:

```
Escobar
Gordon
Odor
Peraza
Swanson
```

If you position exactly 5 cursors in your file and paste, 1 name will be pasted at each cursor location. A GIF is worth a thousand words:

{% include image.html file="2019-04-05-multiple-cursors/multi-replace.gif" alt="GIF that shows multiple words being replaced by different words in a single paste" %}

I’ll leave the steps out — try it for yourself!

> Note: I originally [published this on Medium](https://medium.com/@nafiszaman/multiple-cursors-visual-studio-code-a2e2f531c5b5) a while back. I decided to rewrite it here to keep all of my posts in the same place.