---
layout: post
title: An Easy Subdomain Addressing Bookmarklet for Fastmail
categories: javascript fastmail
---

I've been using [Fastmail as my email provider](https://www.fastmail.com/?STKI=15532514) for a couple of years now, but only recently set up their [subdomain addressing feature](https://www.fastmail.com/help/receive/addressing.html) which allows me to use email addresses like `blah.com@fakeaccount.example.com`. I don't get much spam on my Fastmail account to begin with, but I appreciate the peace of mind that if I do get some I can work out where it came from pretty easily, and it's not as easily stripped as plus address (like `fakeaccount+blah.com@example.com`).

It's not a huge hassle to write these email addresses out by hand, but I noticed that I wasn't doing them consistently and occasionally dropped the TLD or mistyped the domain, so I figured I'd throw together a bookmarklet to autopopulate the email based on the current domain. At first I wanted to write to the clipboard, but it turns out that's still difficult/impossible in JavaScript (for obvious reasons), so I settled on inserting the value in to the currently focused text field instead. Click the bookmarklet and it grabs the current hostname, appends the domain, and away we go.

The code is pretty straightforward, just a few lines:
{% highlight javascript %}
var mail_domain = "mail.example.com";
var hostname = document.createTextNode(window.location.hostname);
document.activeElement.value = hostname.textContent + "@" + mail_domain;
{% endhighlight %}

I [generated a bookmarklet](https://mrcoles.com/bookmarklet/) using the code above, dragged it to my bookmarks bar, and that was it.

All we do is set the subdomain you're using, grab the hostname, then concatenate them and set the value of the `activeElement` on the page to the generated email address. If you've focused on the correct form field you'll be good to go.

You'll also notice that I set up an alias of `mail` at my domain, because I like how the resulting email address looks a lot more than my actual email account, which is `me` at my domain. I'm really happy with the approach so far. The only issue I've had is my own lack of consistency in the email addresses I use, and I'm optimistic that this will take care of that.

There are a few things I'd probably add eventually like stripping off subdomains and raising a warning of some sort if no text field has focus when clicked. I could see potentially wanting to remove the dots from the hostname, if I run in to enough overzealous regular expressions. But I wanted to get this done and blogged about while my kids were napping today, so this will do for now. I figure I'll use it for a while and see how I like it. It was also an excuse to get myself blogging again, as I've been thinking about writing a lot lately but hadn't taken any steps to get back at it.
