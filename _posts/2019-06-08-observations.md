---
layout: post
title: Custom credentials in Ansible Tower to store Github private keys
comments: true
---
This post shows you how to add a custom credential in Ansible Tower, that stores SSH private keys. I won't dive into what custom credentials are and how to get started with them because it is brilliantly covered in this "Inside Playbook" https://www.ansible.com/blog/ansible-Tower-feature-spotlight-custom-credentials by [@bill_nottingham](https://twitter.com/bill_nottingham)
<!--more-->

### My use case

One of the roles in my playbook involves cloning a private git repo. For this, I use the Ansible [git ](https://docs.ansible.com/ansible/latest/modules/git_module.html) module. Since this is a private repo, I cannot use the **https** URL, since doing so, my playbook will always require human input. Running the following playbook:

``` yaml
---
- name: DEMO AN INTERACTIVE PRIVATE KEY
  hosts: localhost
  gather_facts: no
  tasks:

    - name: CLONE THE GIT REPO
      git:
        repo: https://github.com/termlen0/test_repo.git
        dest: /tmp/my_test_repo

```
...will result in the following output:

``` bash
[ec2-user@ip-10-0-0-194 tmp]$ ansible-playbook demo_interactive_private_key.yml 
 [WARNING]: provided hosts list is empty, only localhost is available. Note that the implicit localhost does not
match 'all'


PLAY [DEMO AN INTERACTIVE PRIVATE KEY] *****************************************************************************

TASK [CLONE THE GIT REPO] ******************************************************************************************
Username for 'https://github.com': 



```

...where the playbook is waiting on my interactive response

### Solving using a SSH key pair

This situation is easily remedied by switching from the **https** to a **ssh** url and then using the SSH key pair registered with github.


``` yaml
---
- name: DEMO AN INTERACTIVE PRIVATE KEY
  hosts: localhost
  gather_facts: no
  tasks:

    - name: CLONE THE GIT REPO
      git:
        repo: git@github.com:termlen0/test_repo.git
        dest: /tmp/my_test_repo
        key_file: ./my_private_key

```

### Moving this solution to Ansible Tower

To move this playbook to my central Tower server, I need to take a different approach because I don't want my private key to be on the Tower file system. 
One option is to define a custom credential and supply my private key through this credential. Custom credentials come in handy when you want to inject sensitive data as extra vars or environment variables into your playbook (exhaustive details [here](https://docs.ansible.com/ansible-Tower/latest/html/userguide/credential_types.html))

Let's dive right into it. The work flow I use is as follows:

    1. First we create a new credential type
    
    2. Then, add new credential(s) using this custom credential 

    3. Invoke the credential from the job template

#### 1. Create a new credential type

As a Tower admin, you can create a custom credential by clicking on *ADMINISTRATION> Credential Types* 

[![](/assets/custom_cred1.png)](/assets/custom_cred1.png)

Here, I've created a new credential type and I've named it **PRIVATE KEY CRED**. Custom credentials are defined using 2 components :

    1. Input Configuration
    
    2. Injector Configuration
    
Simply put, the input configuration is where you will define the characteristics/attributes of the custom credential you are defining. In my case:

``` yaml
fields:
  - id: my_private_key
    type: string
    label: private_key
    secret: true
    multiline: true
```

The `secret: true` will make sure that the data stored in this credential will be encrypted at rest and `multiline: true` will accommodate for private keys, which typically contain multiple lines.

The *Injector Configuration* section, is how the data contained in the custom credential will be **injected** into our playbook. Let's break this down for my example:

``` yaml
file:
  template.my_key: "{{ "{{ my_private_key " }} }}"
extra_vars:
  secret_key: "{{ "{{ tower.filename.my_key " }} }}"

```

The `file` parameter tells us that the string data of the `my_private_key` custom credential id will be stored as a temporary file. The `template.my_key` is how Tower will reference this file, using the identifier `my_key`. So now, at this point the data collected from the input field is going to be stored as a temporary file that Tower can reference using `my_key`.

The next part is `extra_vars`. Here, I am configuring the injector to pass the data into the playbook as an extra var. The extra var name is  `secret_key` and the value of this variable will be the temporary file referenced by Tower as `my_key`. This will be clear further down when we debug this variable within our playbook.

> In other words _tower.filename_ is not user defined, whereas *my_key* is.


Ok, now onto using this brand new custom credential that we just created!


#### 2. Add new credentials

This step should look familiar to Tower/AWX users. Create a new credential by choosing the *RESOURCES > Credentials* option from the sidebar:


[![](/assets/custom_cred2.png)](/assets/custom_cred2.png)


> After saving the private key is encrypted and stored securely on Tower. 

Repeat this step for as many users as needed that will be using the github repo.


#### 3. Invoke the custom credential from the job template

This step is where it all comes together. So first, let's update our playbook so that we can take advantage of the custom credential that's being injected as an *extra var*:

``` yaml
---
- name: DEMO AN INTERACTIVE PRIVATE KEY
  hosts: localhost
  gather_facts: no
  tasks:

    - name: DISPLAY THE INJECTED EXTRA VAR
      debug:
        var: secret_key

    - name: CLONE THE GIT REPO
      git:
        repo: git@github.com:termlen0/test_repo.git
        dest: /tmp/my_test_repo
        key_file: "{{ "{{ secret_key "}} }}"
      delegate_to: localhost

```

> Recall that in step 1, we are injecting the private key as a variable named *secret_key*


Great, now let's add a job template that ties this playbook and our custom credential together:

[![](/assets/custom_cred3.png)](/assets/custom_cred3.png)

Note how I have the credential to be **PROMPT ON LAUNCH**? For my use case, I'd like others in my org to be able to interact with this private github repo. Selecting the **PROMPT ON LAUNCH** enables each individual to use their own private ssh key through the custom credentials we discussed above.

When I launch the template, I am prompted to choose a credential type. I choose the custom credential we created **PRIVATE KEY CRED**. Which then lists the available private keys of this type.


[![](/assets/custom_cred4.png)](/assets/custom_cred4.png)

> Use the powerful Tower RBAC to restrict who can use which credentials. 


Finally, launching the template results in the repo being cloned as expected:


[![](/assets/custom_cred5.png)](/assets/custom_cred5.png)


Note the contents of the **secret_key** variable in the play output. It references the temporary file that the custom credential creates (and deletes after the play execution), containing my private key details.

### Conclusion

Hopefully this post helps folks who are trying to figure out how to deal with SSH keys and Github connectivity. Please let me know if you have questions through the comments/social media. 

If you have solved this in Tower using a different method, I'd love to hear how you did it as well. 





