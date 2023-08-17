---
title: "Unit tests for AWS CDK"
date: 2023-08-06T17:08:24+02:00
draft: true
---

---

I recently started using AWS CDK in a project that I'm now contributing to
at work. It is a new experience for me, this tool. I've turned a blind eye
to it for some time as a result of bad experiences in the past. Not with AWS CDK
itself, mind you, but with the concept.

The basic premise of AWS CDK is that you can use the power of a programming
language to describe your infrastructure.

> **AWS Cloud Development Kit (CDK)**
>
> Define your cloud application resources using familiar programming languages
>
> — [AWS CDK documentation](https://aws.amazon.com/cdk/)

And that bit about using programming languages to describe infrastructure was enough
reason for me not to use CDK. That and the use of CloudFormation under the hood. It's
not that I'm not a fan of infrastructure as code. Not at all! I'm all in for it.
But for that, my preference has been Terraform.

My main issue with using programming languages to describe infrastructure is
complexity. Using a programming language makes it easy to create too many
abstractions, which leads to making it very hard to predict what the resulting
infrastructure is going to look like. When I used [Troposphere](https://github.com/cloudtools/troposphere),
a Python library that allows you to more easily create CloudFormation templates,
I had to do some trial and error until I got the right output. Eventually, I learned
the codebase and got used to it and writing code and debugging got faster, but I
still felt that programming languages made it easy to add complexity and that the
simplicity of Terraform was superior to the power of programming languages.
I still hold this thinking, but I also know that AWS CDK has its space for some
use cases and, more importantly, that with unit tests I can mitigate most of
the issues I had in the past.

With unit tests I can shorten my feedback loop and be sure that the changes
I make are exactly what I want them to be. And I believe this is especially
important for people who are trying to contribute their first changes to a new
project. Without having to dive into the codebase and learn all the little bit and pieces,
they can easily test and be confident that the code they've written won't inadvertently
change the desired output. And because I know I'm no expert in AWS CDK, one of
my first contributions to this new project I've jumped into was to add some unit
tests.

## Testing a basic stack


