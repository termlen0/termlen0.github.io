---
layout: post
title: Cleaning up pending Tower Jobs
comments: true
---
This post is a quick troubleshooting gist based on a recent problem I encountered. I had an external system make API calls to Ansible Tower and trigger a job template. Pretty standard stuff...however, the configuration on the remote system was, ummm, not very accurate and I ended up with a ton of API calls to my Tower machine. 

<!--more-->

This caused the jobs to go into a `pending` status and I could no longer trigger any of my normal jobs. I found the following snippet of Python code from https://github.com/ansible/awx/issues/1034 that does the trick!


``` python
from awx.main.models import UnifiedJob
for i in UnifiedJob.objects.filter():
   if(i.status == 'pending'):
     i.delete()
```

### STEPS:

 - In order to invoke this, first switch to a root user (that seems to work for me)
 - Invoke `shell_plus` by issuing the following command `awx-manage shell_plus`. This starts an interactive Python shell.
 - Copy and paste the Python code and wait for all the pending jobs to disappear!
 
 Hope this helps you too!
 
 
