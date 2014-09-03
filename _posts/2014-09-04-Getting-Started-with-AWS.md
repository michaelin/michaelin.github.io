---
layout: post
title: Getting Started with AWS

---

One of the things I'd like to start playing with is [Packer][1], a nifty tool for configuring and building virtual machine images. 

I thought I'd start by going through the [Getting Started guide][2], which is very well written. But when you get to the meat of it, where you are actually going to start building images, you see this little notice at the beginning:

> If you don't have an AWS account, [create one now][3]. For the example, we'll use a "t1.micro" instance to build our image, which qualifies under the AWS free-tier, meaning it will be free. If you already have an AWS account, you may be charged some amount of money, but it shouldn't be more than a few cents.

That statement hides quite a bit of complexity if you are new to AWS. AWS is short for Amazon Web Services, the part of Amazon that offers virtual machine hosting in the Elastic Compute Cloud, also called EC2. Amazon has a very good [getting started guide][4], but there are a lots of options, so why don't I show you how I went about it.

Creating Your Account
=====================

Creating the AWS account is the easy part. Start by going to the [AWS Free Tier overview page][3]. From there you can sign up for an account by jumping through the usual account creation hoops. Have your cell phone at your side, though. Amazon actually checks your cell phone number by calling you.

When you are done with the registration process, you are presented with the AWS Management Console.

![AWS Management Console][img01]

Creating a User
=====================

When working in the cloud, security is not an option but a necessity. If you do not take appropriate steps to secure your machines, it will be the equivalent of always working as root in a linux environment: You will access your instances with the same credentials as you log in to AWS. If one of the machines are compromised, so is your AWS account.

```
TODO: Describe IAM
```

To deal with this, Amazon kindly asks you to set up an IAM user. Start by choosing the IAM under Deployment & Management on the AWS Dashboard. This brings you to the IAM Dashboard.

![AWS IAM Dashboard][img03]

You are met with a security status overview with five recommended tasks:

 * Delete your root access key
 * Activate MFA on your root account
 * Create individual IAM users
 * Use groups to assign permissions
 * Apply an IAM password policy

First order of business is to create a new IAM user, and assign administrator permissions to the user. Creating the user is pretty straight forward, and started on the Users page reached from the menu. Your only choice will be the username of your new user, and whether you want to create more than one user. The most important part of the process is to remember to download the user credentials once the user has been created.

![AWS IAM Download Credentials][img04]

You will need the credentials when you use Packer to create virtual machine instances.

Once the user is created, go to the *Users* page and click on the user you created. At the bottom of the user summary page, in the *Security Credentials* section, you should now assign a password to the user.

To assign access privileges to the user, it must be a member of a user group. Go the the *Groups* page and choose to add a group called *Administrators*. You will be asked to select a permission policy for the group.

![AWS IAM Group Policy][img05]

For this group you should choose the Administator Access policy template. Don't customize it when you are given the option. You can play with that later.

The group has no members when it has been created. Mark the group and select *Add User to Group* from the *Group Actions* list, then add the IAM user you created a few minutes ago.

Now go back to the IAM dashboard to find the IAM users sign-in link. The format of the link is `https://<aws account id>.signin.aws.amazon.com/console`. Sign in with the new user.

You are now well on your way to a secure account. The remaining security warnings in the Security Status on the *IAM Dashboard* can be handled at your own leasure. To sign in with root account credentials, use the link on the sign-in page.

![Sign in as root][img06]

The rest of this post will be using the new administrator account instead of root access.


Creating the instance
===

Now that the user and security is in place, we can create an actual machine instance. Choose the EC2 option in the Services menu to bring up the EC2 Dashboard.

![AWS EC2 Dashboard][img02]

Your first choice is where your instance should be hosted. In the top right corner, you find can find the region selection list. Generally you'll want to host an instance in the region closest to your users, but be aware the [pricing is different for each region][5].

![AWS Region List][img07]

Once you've selected your region, you can push the big blue *Launch Instance* button to get started with creating the instance.

The first step in the wizard asks you to choose an Amazon Machine Image (AMI). If you've worked with local virtual machines before, this can be compared to choosing either an ISO image or a VMware appliance as a starting point for your machine. It will give you anything from a basic OS installation to a fully configured application server, based on your choice of AMI template.

![AWS Choose AMI Template][img08]

You should start with the topmost *Amazon Linux AMI* option for now, or at least one that is *free tier eligible*. You can read more about [root device type][6] and [virtualization type][7] if you want, but it really doesn't matter at this point.

The next step is to choose the instance type you want. This is where you choose the beefiness of your server, i.e. the amount of RAM, number of CPUs etc. There are many options that you can look into at a later times, but only the smallest one called t2.micro is *free tier eligible*. Choose that one, and then press *Review and Launch*.

![AWS Choose Instance Type][img09]

On the *review and launch* page you'll probably be met with a warning stating that your instance will be accessible from any IP address. Unless you have a static IP address, i suggest you leave this alone for now. There are [ways to update dynamic IPs in a security group][8] if your want.

The rest of the review page just lists the choices you've made until now. When you have verified the choices, press *Launch*.

If you think a virtual machine instance is spinning up at this point, you'll probably be disappointed. There are even more steps before we are done.

In order to gain access to your instance, you'll need to create a new [key pair][9].

![AWS Create Key Pair][img10]

The process is straight forward. Give the key pair a name, and download the resulting `.pem` file. If you have cheated, and already created a key pair, you can use that instead of creating a new one.

When you are done, press *Launch Instances*. Magic happens. Gears are churning. Your instance is starting.

Go back to the EC2 Dashboard and choose the Instances option in the menu. You'll see a list of all your instances - that would be one, at this point - and their status. To get instructions on how to connect to the instance, select is and press *Connect*. The method will vary depending on your operating system and your preferences.


[1]: http://packer.io/
[2]: http://www.packer.io/intro/getting-started/
[3]: http://aws.amazon.com/free/
[4]: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html
[5]: http://aws.amazon.com/ec2/pricing/
[6]: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ComponentsAMIs.html
[7]: http://palakonda.org/2012/10/30/aws-virtualization-hvm-vs-paravirtualization/
[8]: http://www.edwiget.name/2013/11/automatically-changing-dynamic-ips-in-aws-security-group/
[9]: http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html


[img01]: ../images/aws-management-console.png "AWS Management Console"
[img02]: ../images/aws-ec2-dashboard.png "AWS EC2 Dashboard"
[img03]: ../images/aws-iam-dashboard.png "AWS IAM Dashboard"
[img04]: ../images/aws-iam-download-credentials.png "AWS IAM Download Credentials"
[img05]: ../images/aws-iam-group-policy.png "AWS IAM Group Policy"
[img06]: ../images/aws-login.png "Sign in as root"
[img07]: ../images/aws-region-list.png "AWS Region List"
[img08]: ../images/aws-choose-ami-template.png "AWS Choose AMI Template"
[img09]: ../images/aws-choose-instance-type.png "AWS Choose Instance Type"
[img10]: ../images/aws-create-key-pair.png "AWS Create Key Pair"