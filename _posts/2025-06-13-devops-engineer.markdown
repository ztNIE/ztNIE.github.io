---
layout: post
title: "I Do a Little Bit of Everything: Life as a DevOps Engineer"
date: 2025-06-13 21:40:43 +1000
categories: blog
---

DevOps, a buzz word you may hear every day now. Along with Agile, it may have become the second most popular and ambiguous term in the Industry. I've been working as a DevOps Engineer for a year now, and I can only depict what "DevOps Engineer" really is by drafting the outlines.

  

## DevOps as a Culture

DevOps as a term is first introduced as a cultural, that "Developer Team" and "Operation Team" should not be separated. Connect them and voila, "DevOps"!

  

In an ideal world of DevOps, there is no separation between Developers and Administrators. People who developed the software will operate the software. There are no communication overhead between two separate departments, you will have a fully functioning team taking care of the full SDLC of the software.

  

So ... What does DevOps Engineer do then? Are you a developer or an administrator?

  

## DevOps Engineer as a Job

  

DevOps Engineer is neither, at least, I'm neither. When people ask me "What do you do as a DevOps engineer", my go-to answer was

  

*"I take care of the things that developers are not bothered to do."*

  

Now, my favourite answer is 

***"I do a little bit of everything."***


That is not answering the question. I know ;)

  

A longer answer would be:

  

There are many gaps between a working prototype running on developers' laptop and a highly-reliable, performant and secure website. My job is to fill the gaps and help the developers to deliver their software in a more reliable, safer and more efficient way.

  

First, we need a place to run the software. We use AWS Elastic Container Service (ECS) to handle the containers.

  

Second, we need to make the application public to the Internet. We use Application Load Balancer (ALB) to expose the APIs and also put a Web Application Firewall (WAF) in front of it to keep it safe.

  

Now, the service is available to the Internet, but we don't like the random hostname AWS gives, we want a pretty subdomain under our company domain, like "service.example.com". We use Cloudflare as our DNS provider.

  

And finally, we IaC it with Terraform, because why not? There is no guarantee of the reliability of the cloud infrastructure if it's configured manually in a web GUI.

  

That's only a part of it. We've got a running application, now we need to operate it -- observability, incident response management, release management, CI/CD, security, and there are more.

  

A DevOps Engineer will need to do a bit of all of these, depending how proficient developers are in each area.

  

Maybe "I do a little bit of everything" is not a bad answer after all ;)

  

## Wait ... That Doesn't Match What I've Heard

  

Yes, that's expected. Every company has its own definition of "DevOps Engineer", what you will do as a DevOps Engineer really depends on what gaps remain. A DevOps engineer in Company A could be fighting CI/CD pipelines everyday, another DevOps engineer in Company B could be all about operating a Kubernetes cluster.

  

It depends.

  

The ambiguity of DevOps engineer comes from the ambiguity of business needs, organization structures -- or really, just the complexity of the world. It's impossible to have a consistent scope for DevOps engineers. We're meant to fill gaps, and every organization has different gaps. It's also hard for DevOps engineer to find the right gap to fill, that's why communication and empathy are also considered essential for DevOps engineer. You need to understand what gaps need to be filled before you work on it.

  

## So ... What do you do again?

  

Sorry that if you've read this blog and still don't know what DevOps engineer does. Nobody knows.

  

We do a little bit of everything.