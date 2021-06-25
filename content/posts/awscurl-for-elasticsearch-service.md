---
title: "awscurl for AWS Elasticsearch Service (via Homebrew)"
date: 2021-06-24T21:27:11-06:00
---

I was recently working on a project that was leveraging Amazon's [Elasticsearch
service](https://aws.amazon.com/elasticsearch-service/). We were fortunate
enough to be able to deploy this in our VPC - which is the recommended way to do
so. The VPC adds a nice security layer, but unfortunately still lets
everything in your VPC talk to your database - which is not following best
security practices. So, to lock this down a little more, we decided to only let
certain user groups talk to the database. This is where we ran into some
headaches with development.

The first thing to note about the IAM access policy method is that you now
need to [sign every
request](https://docs.aws.amazon.com/general/latest/gr/signing_aws_api_requests.html)
to verify the identity of the requester. A relatively simple task at the
application level as there are plenty of SDKs for AWS's elasticsearch, but
becomes a bit of a pain when you want to inspect the DB from your anything
outside your application.

There are two main problems when trying to hit the DB from your local machine.
First, the problem of being in the VPC. This is actually pretty easy to solve;
you can set up a [bastion server](https://en.wikipedia.org/wiki/Bastion_host) in
the VPC with ssh key access. From there you can either port-forward the service to
your local machine or just ssh in to the instance and use the tools there (wget,
curl, ...).

Great, you should be able to talk to your AWS Elasticsearch (ES) Service. Well,
you can't - remember now you need to sign every request with your identity (and
the URL along with the body of the request - so the signature can't just be
computed once, but rather needs to be done for each request individually). AWS
provides many
[scripts/examples](https://docs.aws.amazon.com/general/latest/gr/sigv4-signed-request-examples.html)
for how to do this, but they can be a pain to change for every little request
you want to do (especially because ES makes using curl so easy).

Here comes [`awscurl`](https://github.com/legal90/awscurl) to save the day! This
will allow you to use `curl` like it was normal (well with the `awscurl`
command) and it will sign every request with the given profile. Installing this
and a profile on your bastion host would make for a great solution.

We wanted to be able to do it from our machine though as we didn't want to set
up a bastion server with many dependencies or any special image - plus we wanted
to use our own user accounts to authenticate (sign requests).

We decided to port-forward the ES instance to our local machine and use awscurl
to talk with it. The problem here is that you need to sign the request for the
actual service URL (not localhost). How do you sign requests for one URL but
pass them to localhost? Well that's where my version of
[`awscurl`](https://github.com/bkimmig/awscurl) came in (I should note that I
wanted to mess around with golang a bit, so I split the main file up and broke
it out into a package - I was hoping to add some tests but ran out of time).

How did we use this new version then?

First we port-forwarded our service. Then we added our domain, the url given by
AWS ES service, to our `/etc/hosts` file - mapping it to localhost.

```
127.0.0.1 vpc-your-es-instance.region.es.amazonaws.com
```

Kinda hacky... I know.

From there we can send requests to that URL (sign them for the appropriate URL)
but the port is off. The domain is going to be at port 443, which your computer
likely won't allow you to forward to as it is too low. So you need to sign the
request for `vpc-your-es-instance.region.es.amazonaws.com:443` but your
computer will see it at `vpc-your-es-instance.region.es.amazonaws.com:9200`
(assuming you forwarded to 9200) locally. So now you would use the `--port-map`
flag to say "hey, the request is going to port 9200, but sign it for 443".

```
awscurl --service es --region your-region --profile your-profile --port-map "9200:443" "https://vpc-your-es-instance.region.es.amazonaws.com:9200/"
```

And there ya go! You can now send signed requests to your port-forwarded AWS ES
service.

## Extras

More info on the [example above](https://github.com/bkimmig/awscurl#requests-to-aws-elasticsearch-service-on-a-vpc).

You can see my version of the `awscurl` code [here](https://github.com/bkimmig/awscurl).

If you use `brew` you can install my version of `awscurl` following [these
instructions](https://github.com/bkimmig/homebrew-tools#awscurl).
