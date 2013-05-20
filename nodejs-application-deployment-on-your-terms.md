# Introducing AWSBOX, the DiY PaaS for Node.JS

Once you've written a server in Node.js, how do you deploy it?
Instead of using a pre-existing "Platform as a Service" (PaaS) provider, the [Identity team at Mozilla][] chose to build custom infrastructure atop [Amazon EC2][], and we'd like to tell you more about it.

Meet [awsbox][], a minimalist PaaS layer for Node.js applications that's currently handling nearly two dozen of the non-critical services that we support.
Awsbox was designed to deliver simple, PaaS-style deployment without sacrificing the flexibility of custom infrastructure.

[Identity team at Mozilla]: https://identity.mozilla.com/
[awsbox]: https://github.com/mozilla/awsbox
[Amazon EC2]: http://aws.amazon.com/ec2/

## Using awsbox

In order to deploy a Node.JS project with awsbox, you must make some changes to your code, provide your amazon credentials in the environment, and then you can deploy via the command line.

In terms of code changes, you must:

1. Create an `.awsbox.json` file that specifies how to start the server
2. add `awsbox` as a dependency in your `package.json`
3. ensure your server binds to the port specified in the `PORT` environment variable

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

And now your Node.JS application is hosted and running on an EC2 instance.  At this point, you've spend about twenty minutes with awsbox.  You've made minimal changes to your application to get it deployable.  You've deployed a new server and gotten your application up and running in EC2.  Finally, you've got an easy way to push changes that fits within your existing workflow (you just git push to a remote).

Now that you have a feel for how you use awsbox and the basic features it provides, let's take a step back and look at what it actually is and how it works.

## awsbox is ... A Minimalistic Contract

Any hosting environment has certain expectations of the application that it will be running, *the contract*.  For awsbox this contract includes the following:

**What process(es) should be run** are specified by the app in `.awsbox.json`.

**What software must be installed** is specified by the app in `package.json`. 

**On which port do I communicate with the application** is communicated to the app via the `PORT` environment variable.

In building awsbox, a main goal was minimal invention, to make it easy to "port" an existing application to awsbox.

## awsbox is ... A Machine Image

During the process of creating an instance, awsbox creates a machine instance from an "Amazon Machine Image", which results in a running server that's ready to accept your node.js application and install it.  The image is built from the [Amazon Linux AMI][] which is a custom linux distribution provided by amazon, which has access to popular rpm-based package repositories via [yum][].  The ID of awsbox AMI is referenced in the `awsbox` javascript library.  

[Amazon Linux AMI]: http://aws.amazon.com/amazon-linux-ami/
[yum]: http://en.wikipedia.org/wiki/Yellowdog_Updater,_Modified

This image is *pre-configured with multiple user accounts*.  `ec2-user` is an account that has sudo access to the machine.  `proxy` is an account that hosts an [HTTP reverse proxy][] that with a [few steps][] can serve as an SSL server to let you support HTTPS without modifying your application.  Finally, the `app` user is the account that hosts all of your application code, your server logs, the server based git repository that you push to, and the [git post-commit hook][] responsible for installing dependencies and starting your server after you push.

[HTTP reverse proxy]: http://en.wikipedia.org/wiki/Reverse_proxy
[few steps]: https://github.com/mozilla/awsbox/blob/master/doc/HOW_DO_I.md#how-do-i-enable-ssl
[git post-commit hook]: https://www.kernel.org/pub/software/scm/git/docs/githooks.html

## awsbox is ... Command Line Tools and Libraries

At the time you `npm install` awsbox, a collection of javascript libraries and a command line tool are installed locally.  The command line tool gives you a much faster way to deploy servers than available through Amazon's web console, and handles most of the complexity of creating an instance in EC2 that is ssh and web accessible.

The `awsbox` command line tool also provides many command line *verbs* to perform basic administration of your awsbox, which can be listed with `node_modules/.bin/awsbox -h`.

The most interesting verb is `create`, which actually creates a virtual machine.  

## awsbox is ... A Pile Of Features and Hooks

Finally, any non-trivial server requires more than just a Node.JS service.  To support the unknown awsbox allows you to [specify yum packages that should be installed][] at instance creation time.  For more custom configuration you have two options:

[specify yum packages that should be installed]: https://github.com/mozilla/awsbox/blob/master/doc/JSON.md#packages-optional

**SSH in and install whatever you need to**: The goal of awsbox is to let you move as fast as possible, and sometimes the most expedient way to get a new instance of a service up is to perform them manually and write a README.  But a more repeatable solution is available…

**Write scripts to automatically configure software for you**: Awsbox has the [notion of *hooks*][], which occur at various stages of instance creation or deployment.  Using these hooks, it's possible to [configure mysql][], [install redis manually][], or do whatever you need to in order to get your service running.

[notion of *hooks*]: https://github.com/mozilla/awsbox/blob/master/doc/JSON.md#remote_hooks-optional
[configure mysql]: https://github.com/mozilla/browserid/blob/4971e83b897829d866f99c0e398d52a7b3b9ec2b/scripts/awsbox_remote/post_create.sh
[install redis manually]: https://github.com/mozilla/restmail.net/blob/44306506b1a33ed3c1fbc1b61f13b8d557b80141/aws_scripts/post_create.sh

# Is awsbox for Me?

Having a single consistent mechanism of deploying non-critical services has been an incredible efficiency benefit for our team.  *Collaboration is easier* when you have a simple and well defined contract between application and environment.  *Diagnosis of issues* is faster when you have a consistent set of deployment conventions.  Finally, *moving from experiment to production environment is less costly* when an application has all of its dependencies explicitly expressed.

If you are looking for a deployment solution for your own experimental Node.JS service, we hope the ideas and design of awsbox are useful to you.    
