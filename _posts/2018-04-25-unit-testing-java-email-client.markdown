---
layout: post
title:  "Unit-testing Java Email Client"
date:   2018-04-25 12:18:00 +0200
categories: java email testing
---
This one will be short and will contain mostly links, no single line of code. But following these links is enough to cover the topic. When sending email from Java code is required, good old SMTP and javax.mail package do all the work. What's left for developer if properly align and pass all required parameters.

Good start to read about Java Mail API is probably [here][javamail-oracle]. When it comest to code, is makes sense to go directly to [github][javamail-github]. Usage is pretty straightforward and well described, so I won't stop here (good exampe is [here][aws-smtp-java]). Where I will stop, is how to actually introduce test coverage for email integration code.

It appears that answer to the question "how do I test java email client code" has already been given back in 2011 on [stackoverflow][wiser-answer]. This answer leads, surprise-surprise, to [github][smtp-github], where basic usage of email server code is [described][smtp-server]. And finally, regarding testing, the answer is [given][smtp-wiser] as well. 

That's all folks! Now you see how easy it is to introduce unit-tests for the code that sends emails. Receiving emails is totally different story though and isn't part of this post.

[javamail-oracle]: http://www.oracle.com/technetwork/java/javamail/index.html
[javamail-github]: https://javaee.github.io/javamail/
[aws-smtp-java]: https://docs.aws.amazon.com/ses/latest/DeveloperGuide/send-using-smtp-java.html
[wiser-answer]: https://stackoverflow.com/a/8604585
[smtp-github]: https://github.com/voodoodyne/subethasmtp
[smtp-server]: https://github.com/voodoodyne/subethasmtp/blob/master/UsingSubEthaSMTP.md
[smtp-wiser]: https://github.com/voodoodyne/subethasmtp/blob/master/Wiser.md
