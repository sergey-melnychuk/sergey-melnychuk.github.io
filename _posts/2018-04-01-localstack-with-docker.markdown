---
layout: post
title:  "Testing AWS client with LocalStack"
date:   2018-04-01 14:45:00 +0200
categories: aws localstack docker unit-testing
---
Using AWS SDK always leaves one question open for me - how to introduce test coverage for the integration code. On one side, it doesn't make any sense to test/mock the code outside of your control (unless you're owning AWS SDK code, of course, but I suggest, most probably, you don't). On the other side, leaving code uncovered doesn't really look good as well.

One thing I knew for sure, is that I'm not the first one asking this question, and I indeed wasn't. No better option comes to mind then introduce integration test against 'real' 3rd-party system. Of course, there aren't more real AWS endpoint than the real AWS endpoint, but paying for _each_ run of the test doesn't seem like a wise option to me.

Right next after the real AWS in the list of real AWS endpoints, comes [LocalStack][localstack-home]. It allows you to spin up local endpoints that implement AWS services contracts. Someone finally made their own AWS endpoints and allows you to spin it up at your localhost for free! Installation and available services are very well [described][localstack-github], so I won't cover it here.



[localstack-home]: https://localstack.cloud/
[localstack-github]: https://github.com/localstack/localstack
