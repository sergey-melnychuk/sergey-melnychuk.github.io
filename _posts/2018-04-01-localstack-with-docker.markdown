---
layout: post
title:  "Testing AWS client with LocalStack"
date:   2018-04-01 14:45:00 +0200
categories: aws localstack docker unit-testing
---
Using AWS SDK always leaves one question open - how to add test coverage for the integration code. On one side, it doesn't make any sense to test/mock the code outside of your control (unless you're owning AWS SDK code, of course, but I suggest most probably, you don't).

[localstack-home]: https://localstack.cloud/
[localstack-github]: https://github.com/localstack/localstack
