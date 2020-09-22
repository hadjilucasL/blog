+++
title = "Third Time Lucky"
date = 2020-09-22
[taxonomies]
tags = ["development", "startup", "devops", "testing", "production"]
+++

Today was an exciting day. 

Finally ready to make that big push to production.. for the second time round.
Will our heroes succeed ?

<!-- more -->

9 am on a Tuesday morning. 

With a production deployment window that closes at noon, 
it gave us just about three hours to complete data migration and finally deploy the new
version to prod. This was a big release; new db schema, new api, many bugfixes eagerly
awaiting to appease those forever-open tickets. 

Alas this was our second attempt... 

The first time we soon realised that data migration was incomplete,
half the records had null fields and the code was improperly handling a large chunk of requests. We rushed it.
Even though we tested large parts of the code before, we skipped QA(Quality Assurance) tests 
on those last few pull requests. They were small.. predictable.. why bother ?

And a few hours later we were rolling back production 🐣. 

Post-mortem, on the advice of our CTO, we upped our QA tests. Despite a good unit test coverage, our 
integration tests were lacking. So we put forward an array of QA tests to be done on staging, right before the 
big push to production.  One automating a series of requests to the live system, ensuring the responses are correct, 
one testing out responses based on existing data in the db etc. We applied all the bugfixes, we thoroughly ran all the 
QA tests. It was a bulletproof plan.

So it begins.. Attempt number two.

Everything was going perfectly. The data was migrated, records looked clean and tidy. 
We pushed the big green merge to prod button. A few minutes later we were nervously monitoring elastic beanstalk 
switching out to the new environment. And then things became green! Success 🦅! 

Not so fast shout the deployment deities ! 

A teammate sent off a request as a final test. The system usually takes a few seconds to respond but this time it was
taking longer.. Uh oh -.-' Slack messages started to pop in.. QA: "Prod is not handling any requests" 😱. 

Another hurried rollback follows.

But what happened ? Why even after so many unit tests, integration tests, qa testing on staging did we again manage
to deploy a lemon to prod ? A bit of investigation in the logs and we soon found out. 

Our staging and production were exact clones, or so we thought ! 
The infrastructure was the same with the exception of the db. On production we had replica sets, on staging there were none. 
Running this particular db with replicas was expensive and as staging doesn't really hold anything of significance 
or high demand we thought there was no need to have them. This was the culprit however. 
Our code was not tested in the realm of replicas, and the small asynchronicity in the writes from primary to replicas 
meant that if we tried to read the data immediately after a write, there was a chance that this had not yet been 
commited to the replica we were reading from. 

Knowing the cause, we soon applied a hotfix, QA'ed and for the third time round deployed to prod. 
A few nail biting moments later 🤞 and we were in business. Requests were coming finally.
We did it ! We deployed to prod !

Takeaway message? 

Your production and staging should be a mirror of each other, otherwise you will end up debugging on prod.
Infrastructure, db configuration, scaling tactics, even data if possible ! Everything !  

Cost is too high ? Bring it up, test and the tear down.

Solid testing needs to be coupled with a good staging environment. It is the only way you can confidently 
deploy to production.

![Dino-Mite](dino-mite.jpg)
