# Introducing AWSBOX, the DiY PaaS for Node.JS

Once you've written a server in Node.JS, how do you deploy it?  There are tens of "Platform As A Service" providers to choose from, or you can choose to build up your own hosting platform from an operating system image running on physical or virtual hardware.  Rather than enumerating the tradeoffs to consider, this post presents what we in the [Identity team at Mozilla][] have chosen for about 20 of the non-critical services we support.

Meet [awsbox][], a minimalistic "platform as a service" layer for Node.JS applications built on top of [Amazon EC2][].  The project is open source and consists of a machine image pre-configured to run Node.JS applications, and a command line interface for server creation and management.

[Identity Team At Mozilla]: https://identity.mozilla.com/
[awsbox]: https://github.com/mozilla/awsbox
[Amazon EC2]: http://aws.amazon.com/ec2/

## Using awsbox

In order to deploy a Node.JS project with awsbox, you must make some changes to your code, provide your amazon credentials in the environment, and then you can deploy via the command line.

In terms of code changes, you must:

1. Create an `.awsbox.json` file that specifies how to start the server
2. add `awsbox` as a dependency in your `package.json`
3. ensure your server binds to the port specified in the `PORT` environment variable

**XXX: Shall I include code here?  A diff of the actual delta that this represents?**

To provide your amazon credentials, you must set `AWS_ID` and `AWS_SECRET` in your environment, two values which you can attain through the [amazon management console][].

[amazon management console]: http://aws.amazon.com/console/

With the initial application configuration complete, you can `npm install` which will install awsbox, and you're ready to create your first machine:

    $ node_modules/.bin/awsbox create -n MyFirstAWSBOX
    reading .awsbox.json
    attempting to set up VM "MyFirstAWSBOX"
       ... VM launched, waiting for startup (should take about 20s)
       ... Instance ready, setting human readable name in aws
       ... name set, waiting for ssh access and configuring
       ... public url will be: http://<IP ADDRESS>
       ... nope.  not yet.  retrying.
       ... victory!  server is accessible and configured
       ... applying system updates
       ... and your git remote is all set up
       ... configuring SSL behavior (enable)
    
    Yay!  You have your very own deployment.  Here's the basics:
    
     1. deploy your code:  git push MyFirstAWSBOX HEAD:master
     2. visit your server on the web: http://<IP ADDRESS>
     3. ssh in with sudo: ssh ec2-user@<IP ADDRESS>
     4. ssh as the deployment user: ssh app@<IP ADDRESS>

The final step to deploy your application is to `git push`:

    $ git push MyFirstAWSBOX HEAD:master

And now your Node.JS application is hosted and running on an EC2 instance.  At this point, you've spend about twenty minutes with awsbox.  You've made minimal changes to your application to get it deployable.  You've deployed a new server and gotten your application up and running in EC2.  You've got an easy way to push changes that fits within your existing workflow (you just git push to a remote).

Now that you have a feel for how you use awsbox and the basic features it provides, let's take a step back and look at what it actually is and how it works.

## AWSBOX is ... A Minimalistic Contract

Any hosting environment has certain expectations of the application that it will be running, the contract.  For awsbox this contract includes the following from an app:

**What process(es) should be run** are specified in `.awsbox.json`.

**What software must be installed** is specified in `package.json`. 

**On which port do I communicate with the application** is communicated via the `PORT` environment variable.



## AWSBOX is ... A Machine Image

## AWSBOX is ... Command Line Tools and Libraries

## AWSBOX is ... A Pile Of Features and Hooks

# Is AWSBOX for Me?

XXX:  Let's reel it back in and keep it real.  This is a solution that's
XXX:  Worked fantastic for our team, how else could a handful of folks
XXX:  support 20 distinct alpha/beta services?

XXX:  the bottom line is...
