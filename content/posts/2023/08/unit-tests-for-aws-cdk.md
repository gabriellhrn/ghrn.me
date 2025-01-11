---
title: "Unit tests for AWS CDK"
date: 2023-08-20T16:13:00+02:00
lastmod: 2023-09-04T13:41:00+02:00
draft: false
categories: ['tech']
tags: ['aws', 'cdk', 'testing', 'go', 'terraform', 'cloud', 'tutorial']
---

I recently started using AWS CDK in a project that I'm now contributing to at
work. It is a new experience for me, this tool. I've turned a blind eye to it
for some time as a result of bad experiences in the past. Not with AWS CDK
itself, mind you, but with the concept.

The basic premise of AWS CDK is that you can use the power of a programming
language to describe your infrastructure.

> **AWS Cloud Development Kit (CDK)**
>
> Define your cloud application resources using familiar programming languages
>
> — [AWS CDK documentation](https://aws.amazon.com/cdk/)

And that bit about using programming languages to describe infrastructure was
enough reason for me not to use CDK. That and the use of
[CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html)
under the hood. It's not that I'm not a fan of infrastructure as code. Not at
all! I'm all in for it. But for that, my preference has been Terraform.

<!-- vale off -->
My main issue with using programming languages to describe infrastructure is
complexity. Using a programming language makes it easy to create too many
abstractions, which leads to making it very hard to predict what the resulting
infrastructure is going to look like. When I used
[Troposphere](https://github.com/cloudtools/troposphere), a Python library that
allows you to more easily create CloudFormation templates, I had to do some
trial and error until I got the right output. Eventually, I learned the codebase
and got used to it and writing code and debugging got faster, but I still felt
that programming languages made it easy to add complexity and that the
simplicity of Terraform was superior to the power of programming languages. I
still hold this thinking, but I also know that AWS CDK has its place for some
use cases and, more importantly, that with unit tests I can mitigate most of the
issues I had in the past.
<!-- vale on -->

With unit tests I can shorten my feedback loop and be sure that the changes I
make are exactly what I want them to be. And I believe this is especially
important for people who are trying to contribute their first changes to a new
project. Without having to dive into the codebase and learn all the little bits
and pieces, they can easily test and be confident that the code they've written
won't inadvertently change the desired output. And because I know I'm no expert
in AWS CDK, one of my first contributions to this new project I've jumped into
was to add some unit tests.

## Testing a basic stack

For this post, I created a sample stack using the AWS CDK library for Go to
illustrate how to write unit tests for CDK. The full code is available in [this
repository](https://github.com/gabriellhrn/aws-cdk-example). AWS CDK supports
multiple programming languages, but from this point on my examples will focus on
Go.

We test CDK stacks with a package called
[`assertions`](https://pkg.go.dev/github.com/aws/aws-cdk-go/awscdk/v2/assertions).
This package focuses on asserting tests against CloudFormation templates. That's
how CDK works internally. The framework allows you to write code using a
programming language, like Go, and when you deploy, it transforms the Go code
into CloudFormation templates and provisions the infrastructure defined in the
templates using CloudFormation. We don't need to test the provisioning part —
that's AWS's job. As for us, our job is to ensure that our CloudFormation templates
are valid and have the resources and properties they're supposed to have.
Accordingly, asserting the templates generated from our code guarantees the
correctness of any changes we might have made to the code. Let's jump into the
nitty-gritty of this thing then.

In the version
[`v0.1.0`](https://github.com/gabriellhrn/aws-cdk-example/tree/v0.1.0) of the
sample stack, the infrastructure is composed of 3 resources: an SNS Topic, an
SQS Queue and an S3 Bucket. For this first version, we also want the queue to be
subscribed to the topic. We start our
[test](https://github.com/gabriellhrn/aws-cdk-example/blob/v0.1.0/app/app_test.go)
by initializing an app and a stack. An app is the application written with Go to
define the AWS infrastructure. The app works as a container for one or more
stacks. Stacks are the unit of deployment; they're stacks as in CloudFormation
stacks. All resources in a stack are deployed as a single unit by
CloudFormation.

Our first assertion then checks that exactly one of each resource is present in
the template:

```go
func TestAppStack(t *testing.T) {
	app := awscdk.NewApp(nil)
	id := "test"
	stack := NewAppStack(app, id, nil)

	template := assertions.Template_FromStack(stack, nil)
	template.ResourceCountIs(jsii.String("AWS::SNS::Topic"), jsii.Number(1))
	template.ResourceCountIs(jsii.String("AWS::SQS::Queue"), jsii.Number(1))
	template.ResourceCountIs(jsii.String("AWS::S3::Bucket"), jsii.Number(1))
}
```

Note that we're only testing the function `NewAppStack()`. If we had other
functions, we should test them as well. You will also notice that we're using [a
package called `jsii`](https://github.com/aws/jsii). Without going into much
details, AWS CDK is written in JavaScript and `jsii` is the technology that
enables other languages to naturally interact with this JavaScript code.

Running this test will return an error if our resources have not been defined in
code yet:

```bash
$ make test
=== RUN   TestAppStack
--- FAIL: TestAppStack (4.02s)
panic: Error: Expected 1 resources of type AWS::SNS::Topic but found 0 [recovered]
        panic: Error: Expected 1 resources of type AWS::SNS::Topic but found 0
[...]
FAIL
make: *** [Makefile:9: test] Error 1
```

To make this test pass, we need to [create one of each
resource](https://github.com/gabriellhrn/aws-cdk-example/blob/v0.1.0/app/app.go)
the test wants:

```go
func NewAppStack(scope constructs.Construct, id string, props *AppStackProps) awscdk.Stack {
	var sprops awscdk.StackProps
	if props != nil {
		sprops = props.StackProps
	}
	stack := awscdk.NewStack(scope, &id, &sprops)

	awss3.NewBucket(stack, jsii.String("AppBucket"), &awss3.BucketProps{})

	awssqs.NewQueue(stack, jsii.String("AppQueue"), &awssqs.QueueProps{
		VisibilityTimeout: awscdk.Duration_Seconds(jsii.Number(300)),
	})

	awssns.NewTopic(stack, jsii.String("AppTopic"), &awssns.TopicProps{})

	return stack
}
```

Running the test again yields a success:

```bash
$ make test
=== RUN   TestAppStack
--- PASS: TestAppStack (4.21s)
PASS
ok      app     4.224s
```

<!-- vale off -->
But this stack is not complete yet. I also want to subscribe the SQS Queue to
the SNS Topic. The subscription in CloudFormation is a separate resource with a
reference to the queue, so we need to add a new `ResourceCountIs()` assertion
for the new resource:
<!-- vale on -->

```go
	template.ResourceCountIs(jsii.String("AWS::SNS::Subscription"), jsii.Number(1))
```

We also want to make sure that the queue is subscribed to the topic. For that we need to
use two new functionalities provided by the `assertions` package, [capture](https://pkg.go.dev/github.com/aws/aws-cdk-go/awscdk/v2/assertions#readme-capturing-values) and
asserting [resource properties](https://pkg.go.dev/github.com/aws/aws-cdk-go/awscdk/v2/assertions#readme-resource-matching-retrieval):

```go
	subscriptionCapture := assertions.NewCapture(assertions.Match_ObjectLike(
		&map[string]interface{}{
			"Fn::GetAtt": []string{
				"AppQueueXXXXXXXX",
				"Arn",
			},
		},
	))

	template.HasResourceProperties(jsii.String("AWS::SNS::Subscription"), map[string]interface{}{
		"Protocol": "sqs",
		"Endpoint": subscriptionCapture,
	})
```

We use `NewCapture()` and the matcher API (`Match_ObjectLike`) to capture a
value from the template. In this case, we want to capture the reference to the
queue. In essence, the
[ARN](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference-arns.html) of
the queue that will be used as the value for the parameter `Endpoint` of the
SNS Subscription resource. Likewise, we use the method `HasResourceProperties` to
check that the subscription has the ARN of the queue (the captured value) as its
endpoint.

The test will fail again because we haven't created the subscription, so we
update the code to reflect what we want in the stack:

```go
func NewAppStack(scope constructs.Construct, id string, props *AppStackProps) awscdk.Stack {
	var sprops awscdk.StackProps
	if props != nil {
		sprops = props.StackProps
	}
	stack := awscdk.NewStack(scope, &id, &sprops)

	awss3.NewBucket(stack, jsii.String("AppBucket"), &awss3.BucketProps{})

	queue := awssqs.NewQueue(stack, jsii.String("AppQueue"), &awssqs.QueueProps{
		VisibilityTimeout: awscdk.Duration_Seconds(jsii.Number(300)),
	})

	topic := awssns.NewTopic(stack, jsii.String("AppTopic"), &awssns.TopicProps{})
	topic.AddSubscription(awssnssubscriptions.NewSqsSubscription(
		queue,
		&awssnssubscriptions.SqsSubscriptionProps{},
	))

	return stack
}
```

And now, if we run the tests again, we'll see…

```bash
$ make test
=== RUN   TestAppStack
--- FAIL: TestAppStack (4.06s)
panic: Error: Template has 1 resources with type AWS::SNS::Subscription, but none match as expected.
The 1 closest matches:
AppQueuetestAppTopicDFBA806E18B28849 :: {
  "DependsOn": [ "AppQueuePolicyBD4F9387" ],
  "Properties": {
    "Endpoint": {
      "Fn::GetAtt": [
!!       Expected AppQueueXXXXXXXX but received AppQueueFD3F4958
        "AppQueueFD3F4958",
        "Arn"
      ]
    },
    "Protocol": "sqs",
    "TopicArn": { "Ref": "AppTopic115EA044" }
  },
  "Type": "AWS::SNS::Subscription"
} [recovered]
[...]
FAIL    app     4.072s
FAIL
make: *** [Makefile:9: test] Error 1
```

<!-- vale off -->
That they fail!? Yes, they fail because we were expecting a queue with the
resource name `AppQueueXXXXXXXX`, but we only have one with the name
`AppQueueFD3F4958`. This name is not actually the name of the resource. This is
a [logical
ID](https://docs.aws.amazon.com/cdk/v2/guide/identifiers.html#identifiers_logical_ids)
generated by CDK. The number is an 8-digit hash that CDK generates based on the
name of the stack and resource (that second argument that we pass to the methods
when we create a stack or a resource). Since this hash is based on the values just
mentioned, it's safe to add it to the test: there's more chances that we're
going to modify the properties of this resource rather than its name; and if we
change its name, we should reflect it in the test as well.
<!-- vale on -->

After updating the capture, our test now looks like this:

```go
func TestAppStack(t *testing.T) {
	app := awscdk.NewApp(nil)
	id := "test"
	stack := NewAppStack(app, id, nil)

	template := assertions.Template_FromStack(stack, nil)
	template.ResourceCountIs(jsii.String("AWS::SNS::Subscription"), jsii.Number(1))
	template.ResourceCountIs(jsii.String("AWS::SNS::Topic"), jsii.Number(1))
	template.ResourceCountIs(jsii.String("AWS::SQS::Queue"), jsii.Number(1))
	template.ResourceCountIs(jsii.String("AWS::S3::Bucket"), jsii.Number(1))

	subscriptionCapture := assertions.NewCapture(assertions.Match_ObjectLike(
		&map[string]interface{}{
			"Fn::GetAtt": []string{
				"AppQueueFD3F4958",
				"Arn",
			},
		},
	))

	template.HasResourceProperties(jsii.String("AWS::SNS::Topic"), map[string]interface{}{})
	template.HasResourceProperties(jsii.String("AWS::SNS::Subscription"), map[string]interface{}{
		"Protocol": "sqs",
		"Endpoint": subscriptionCapture,
	})

	template.HasResourceProperties(jsii.String("AWS::SQS::Queue"), map[string]interface{}{
		"VisibilityTimeout": 300,
	})

	template.HasResourceProperties(jsii.String("AWS::S3::Bucket"), map[string]interface{}{})
}
```

And our tests return success again:

```bash
$ make test
=== RUN   TestAppStack
--- PASS: TestAppStack (4.22s)
PASS
ok      app     4.230s
```

Now we have the basic stack we wanted from the beginning. If we want to extend
this code in the future, we now have more confidence to do it because we know
that the tests will yield errors if the resources we just created are missing.

## Extending the stack

Now I need to extend my infrastructure. I want to create another SQS Queue and a
new S3 Bucket. But this time, these resources are slightly different from what
was created before. I want the queue to be a
[FIFO](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues.html)
queue and the bucket is supposed to host a static website. I want to refactor my
code to make it easier to create the buckets and the queues, but I know that I
can't change the resources I already have. And that's exactly how I can use my
unit tests to make my code changes predictable.

This time, I'm going to focus first on the new queue and later on the bucket. We
start similarly to next time, by updating the queue count to 2. But we want two
different queues, one standard and the other a FIFO queue. In that case, we also
need to update our test to make sure we have both types of queues:

```go
	template.ResourceCountIs(jsii.String("AWS::SQS::Queue"), jsii.Number(2))

    template.HasResourceProperties(jsii.String("AWS::SQS::Queue"), map[string]interface{}{
		"FifoQueue": true,
	})

	template.HasResourceProperties(jsii.String("AWS::SQS::Queue"), map[string]interface{}{
		"FifoQueue": assertions.Match_Absent(),
	})
```

Note how we're using the method `HasResourceProperties()` to match each queue.
For the FIFO queue, we set the parameter `FifoQueue` to `true`. For the standard
queue we use the matcher API to tell the `assertions` package that we want a
queue that **does not** have the parameter `FifoQueue`. Since we never set any
value for this parameter for the standard queue, then it should not show up in
the CloudFormation template at all.

Now we can refactor our code slightly, to simplify queue creation and allow us
to create FIFO queues:

```go
func NewAppStack(scope constructs.Construct, id string, props *AppStackProps) awscdk.Stack {
	var sprops awscdk.StackProps
	if props != nil {
		sprops = props.StackProps
	}
	stack := awscdk.NewStack(scope, &id, &sprops)

	createQueue(stack, "AppQueue", nil)
	createFifoQueue(stack, "FifoQueue", nil)

	return stack
}

func createQueue(stack awscdk.Stack, id string, props *awssqs.QueueProps) awssqs.Queue {
	return awssqs.NewQueue(stack, jsii.String(id), props)
}

func createFifoQueue(stack awscdk.Stack, id string, props *awssqs.QueueProps) awssqs.Queue {
	if props == nil {
		props = &awssqs.QueueProps{}
	}

	props.Fifo = jsii.Bool(true)

	return awssqs.NewQueue(stack, jsii.String(id), props)
}
```

The example above omits the other resources so we can focus on the queues, but
if we ran the tests, we would see that they return a success, because the old
queue was generated with the same properties, despite the refactoring.

We can go ahead and do the same for the bucket now. We increase the resource
count to 2 for the bucket and add a new assertion for the resource properties.
This new bucket will be used for static hosting, so we need to check if it
contains the property `WebsiteConfiguration` correctly configured. We'll also
make sure that the existing bucket won't have its properties changed or, in
other words, will have the property `WebsiteConfiguration` absent:

```go
	template.ResourceCountIs(jsii.String("AWS::S3::Bucket"), jsii.Number(2))

	template.HasResourceProperties(jsii.String("AWS::S3::Bucket"), map[string]interface{}{
		"WebsiteConfiguration": assertions.Match_Absent(),
	})

	template.HasResourceProperties(jsii.String("AWS::S3::Bucket"), map[string]interface{}{
		"WebsiteConfiguration": map[string]string{
			"IndexDocument": "index.html",
		},
	})
```

Similarly to what we did for the queues, we'll refactor the code a bit to
simplify the creation of standard buckets and buckets for static hosting:

```go
func NewAppStack(scope constructs.Construct, id string, props *AppStackProps) awscdk.Stack {
	var sprops awscdk.StackProps
	if props != nil {
		sprops = props.StackProps
	}
	stack := awscdk.NewStack(scope, &id, &sprops)

	createBucket(stack, "AppBucket", nil)
	createWebsiteBucket(stack, "WebsiteBucket", nil)

	return stack
}

func createBucket(stack awscdk.Stack, id string, props *awss3.BucketProps) awss3.Bucket {
	return awss3.NewBucket(stack, jsii.String(id), props)
}

func createWebsiteBucket(stack awscdk.Stack, id string, props *awss3.BucketProps) awss3.Bucket {
	if props == nil {
		props = &awss3.BucketProps{
			WebsiteIndexDocument: jsii.String("index.html"),
			PublicReadAccess:     jsii.Bool(true),
		}
	}

	return createBucket(stack, id, props)
}
```

We should also test the Bucket Policy, since we want the website bucket to be
public. I'm not going to cover it here, but the [full
code](https://github.com/gabriellhrn/aws-cdk-example/blob/v0.2.0/app/app_test.go#L71-L73)
has the assertion.

The full test after all the changes look like this now:

```go
func TestAppStack(t *testing.T) {
	app := awscdk.NewApp(nil)
	id := "test"
	stack := NewAppStack(app, id, nil)

	template := assertions.Template_FromStack(stack, nil)

	template.ResourceCountIs(jsii.String("AWS::SNS::Topic"), jsii.Number(1))
	template.ResourceCountIs(jsii.String("AWS::SNS::Subscription"), jsii.Number(1))
	template.ResourceCountIs(jsii.String("AWS::SQS::Queue"), jsii.Number(2))
	template.ResourceCountIs(jsii.String("AWS::S3::Bucket"), jsii.Number(2))
	template.ResourceCountIs(jsii.String("AWS::S3::BucketPolicy"), jsii.Number(1))

	subscriptionCapture := assertions.NewCapture(assertions.Match_ObjectLike(
		&map[string]interface{}{
			"Fn::GetAtt": []string{
				"AppQueueFD3F4958",
				"Arn",
			},
		},
	))

	template.HasResourceProperties(jsii.String("AWS::SNS::Topic"), map[string]interface{}{})

	template.HasResourceProperties(jsii.String("AWS::SNS::Subscription"), map[string]interface{}{
		"Protocol": "sqs",
		"Endpoint": subscriptionCapture,
	})

	template.HasResourceProperties(jsii.String("AWS::SQS::Queue"), map[string]interface{}{
		"FifoQueue": true,
	})

	template.HasResourceProperties(jsii.String("AWS::SQS::Queue"), map[string]interface{}{
		"FifoQueue":         assertions.Match_Absent(),
		"VisibilityTimeout": 300,
	})

	template.HasResourceProperties(jsii.String("AWS::S3::Bucket"), map[string]interface{}{
		"WebsiteConfiguration": assertions.Match_Absent(),
	})

	bucketPolicyCapture := assertions.NewCapture(assertions.Match_ObjectLike(
		&map[string]interface{}{
			"Ref": "WebsiteBucket75C24D94",
		},
	))

	template.HasResourceProperties(jsii.String("AWS::S3::Bucket"), map[string]interface{}{
		"WebsiteConfiguration": map[string]string{
			"IndexDocument": "index.html",
		},
	})

	template.HasResourceProperties(jsii.String("AWS::S3::BucketPolicy"), map[string]interface{}{
		"Bucket": bucketPolicyCapture,
	})
}
```

And this is the code after the refactoring and the inclusion of new resources:

```go
func NewAppStack(scope constructs.Construct, id string, props *AppStackProps) awscdk.Stack {
	var sprops awscdk.StackProps
	if props != nil {
		sprops = props.StackProps
	}
	stack := awscdk.NewStack(scope, &id, &sprops)

	createBucket(stack, "AppBucket", nil)
	createWebsiteBucket(stack, "WebsiteBucket", nil)

	queue := createQueue(stack, "AppQueue", &awssqs.QueueProps{
		VisibilityTimeout: awscdk.Duration_Seconds(jsii.Number(300)),
	})

	topic := awssns.NewTopic(stack, jsii.String("AppTopic"), &awssns.TopicProps{})
	topic.AddSubscription(awssnssubscriptions.NewSqsSubscription(
		queue,
		&awssnssubscriptions.SqsSubscriptionProps{},
	))

	createFifoQueue(stack, "FifoQueue", nil)

	return stack
}

func createBucket(stack awscdk.Stack, id string, props *awss3.BucketProps) awss3.Bucket {
	return awss3.NewBucket(stack, jsii.String(id), props)
}

func createWebsiteBucket(stack awscdk.Stack, id string, props *awss3.BucketProps) awss3.Bucket {
	if props == nil {
		props = &awss3.BucketProps{
			WebsiteIndexDocument: jsii.String("index.html"),
			PublicReadAccess:     jsii.Bool(true),
		}
	}

	return createBucket(stack, id, props)
}

func createQueue(stack awscdk.Stack, id string, props *awssqs.QueueProps) awssqs.Queue {
	return awssqs.NewQueue(stack, jsii.String(id), props)
}

func createFifoQueue(stack awscdk.Stack, id string, props *awssqs.QueueProps) awssqs.Queue {
	if props == nil {
		props = &awssqs.QueueProps{}
	}

	props.Fifo = jsii.Bool(true)

	return awssqs.NewQueue(stack, jsii.String(id), props)
}
```

And if I were to run the tests again, we would see as a result…

```bash
$ make test
=== RUN   TestAppStack
--- PASS: TestAppStack (4.01s)
PASS
ok      app     4.025s
```

A success! We can stage our changes, commit them and open our PR.

## Final thoughts

The examples I used in this post lack some important pieces of code, but the
repository contains a working application. If you want to try this out, you can
clone the repository and follow the instructions in the
[README](https://github.com/gabriellhrn/aws-cdk-example/tree/main). There's a
Dockerfile in the repository to create an image with all tools required to run
the examples, so the only requirement for this to work is that you have Docker
installed.

---

When I started writing code, more than a decade ago, I wasn't a big fan of
writing tests. It took me a while to understand their importance. And I
understand that testing infrastructure is particularly hard and expensive. In
this post I focused on unit tests, but they can't catch all the issues by
themselves. Testing has multiple layers. But even with multiple layers, we can
never have full trust that we're not introducing bugs. However, the point of
testing is to describe how you want your software to behave and make sure that
over time it behaves as expected. It's easy to maintain all the context of a
small codebase in your head, but once you start taking care of multiple projects
and accumulating responsibilities, to err is just human. So, if a bug happened
once, catch that in a test and don't let it happen again.

The shame, really, is to get fooled twice.

---
