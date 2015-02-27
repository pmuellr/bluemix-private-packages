bluemix-private-packages
================================================================================

This project shows how to use "private" node packages in your
[Bluemix][bluemix] app.
The technique will also work with any [Cloud Foundry][cloud-foundry]-based PaaS.

The project is based on the super-simplistic [bluemix-hello-node][] sample
application; check that app if you're curious about the basics of building
a node app for Bluemix.



what is a "private" package?
================================================================================

For purposes of this example, a "private" package is a node package not stored
at the package repository used when you run `npm install`.  By default, that's
the one at <http://npmjs.org> (but
[you can use a different one][strongloop-npm]).

Why would you use a private package in the first place?  Perhaps you have
packages which you don't want to make public ever, or at least yet.  Or maybe
the packages aren't secret, but are things that aren't generally useful for
other people, and so there's no real need to put the packages in a
public npm repository.

node.js already has ways of dealing with private packages.  People generally
use one of the following approaches:

* The [`npm link`][npm-link] command allows you to create a link to a
  private package which exists somewhere else on your development box,
  into your project's `node_modules` directory.

* The `npm link` command can be fiddly, so some folks just use plain old
  [symbolic links][sym-links] to link a private package somewhere else on your
  development box,
  into your project's `node_modules` directory.

* The [`npm` URL dependency mechanism][npm-url-dep] can be used to reference
  your private packages stored in a private git repo or web server.

None of these techniques is going to work well for a node app that you
want to run on Bluemix, or any Cloud Foundry based PaaS.  The basic problem is
that the private packages you are storing outside of your project aren't going
to be visible to the staging machine when you run `cf push` to upload your app.
In the case of using the `npm` URL dependency mechanism, the staging machine may
not have access to your private git repo/web server (eg, if it's behind
a firewall).

This project shows one way of dealing with the problem.  The trick is that
we maintain the private packages in a directory in the project, but not in
the usual `node_modules` directory.  We'll maintain those private packages
using the `npm` URL dependency mechanism, but we'll only ever update them
on our development box, and arrange to have those modules uploaded when
you run `cf push`. Finally, we'll add an `npm` [postinstall script][npm-scripts]
to your project's `package.json`, which will copy the packages from the
separately maintained private package directory in the project, into the
`node_modules` directory.



demo / explanation
================================================================================

In the sample app, what we'll be showing is referencing dependencies of an
application as a git url - in this specific case, it's pointing to a git url
at GitHub, but this story will work fine for git repos you might have behind
your network's firewall, or even on your own machine.

To get started with the sample app provided in this project,
open a terminal session on your development box, and run:

    git clone https://github.com/pmuellr/bluemix-private-packages.git

And then run:

    cd bluemix-private-packages

To install all of the dependencies, including the private ones, run:

    npm install

In the middle of the `npm install` output, you should see something like this:

    ...
    > node node_modules_private/post-install

    post-install.js: running `npm install`
    underscore@1.6.0 node_modules/underscore
    
    ncp@2.0.0 node_modules/ncp
    
    cfenv@0.2.0 node_modules/cfenv
    +-- ports@1.1.0
    +-- js-yaml@3.0.2 (esprima@1.0.4, argparse@0.1.15)

    ...
    moar npm output here
    ...

    post-install.js: done
    ...

What is that `post-install.js` thing?  It's an `npm`
[postinstall script][npm-scripts], which gets run whenever you run
`npm install`.

The `post-install.js` script is does the following:

* changes the current directory to `node_modules_private` - this is the
  directory we are using to manage our private packages

* runs `npm install` in that directory - this will install the private
  packages based on the `node_modules_private/package.json` file, which uses `npm` URL dependencies to manage the private packages

* uses [npc](https://www.npmjs.com/package/ncp) to copy the packages it just `npm install`'d, into the main project's `node_modules` directory

At this point, you can run the app by running

    npm start

You can also push the app to Bluemix, by running

    cf push

When the application is pushed to Bluemix, an `npm install` will get run.
This means the `npm` postinstall script will be run, but it works slightly
different on a Bluemix staging machine compared to when you run it locally.
Specifically, it does **NOT** run `npm install` in your `node_modules_private`
directory.  It doesn't run it, because it would fail, if your URL dependencies
pointed to a git repo or web site that is not on the public internet.

So, in order to make this work, you need to do one additional thing, and that
is to arrange to upload your `node_modules_private/node_modules` directory when
you run `cf push`.  If you use a `.cfignore` file in your project to exclude
the `node_modules` directory - so you don't have to upload all of your
node modules with your app when you run `cf push` - this will also cause
the `node_modules_private/node_modules` to be ignored.  

There's a tiny change to your `.cfignore` file which will fix this.  If
your `.cfignore` file looks like this:


    node_modules
    tmp

change it to this:

    /node_modules
    tmp

That will cause **JUST** the top-level node_modules directory to be excluded
from the upload during a `cf push`, and allow the
`node_modules_private/node_modules` directory to be uploaded.



how to use this in your own project
================================================================================

What we'll be doing is keeping our private packages in a new subdirectory,
`node_modules_private`.  That directory has a `package.json` file, with the
dependencies for our private packages.  The contents for this example are:

```js
{
  "name": "node_modules_private",
  "version": "0.0.0",
  "dependencies": {
    "ncp": "2.0.0",
    "cfenv":      "git://github.com/cloudfoundry-community/node-cfenv.git#0.2.0",
    "underscore": "*"
  }
}
```

The URL for the `cfenv` package points to a GitHub repo, and the `commit-ish`
fragment identifies a particular tag.  I included `underscore` just to make
sure the code works with more than one package `:-)` Please note that you should not remove the `ncp` package because it is used to support recursive copying on Windows machines as well as Unix-based systems. It will not be copied into your `/node_modules` directory.

You should also copy the `node_modules_private/post-install.js` script
into your `node_modules_private` directory.

In your project's main `package.json`, you'll want to set up the
postinstall script.  The postinstall script shows up in the file
as the `scripts.postinstall` property, as in the following example:

```js
{
  "name":           "my-awesome-app",
  "scripts": {
    "start":        "node server",
    "postinstall":  "node node_modules_private/post-install"
  },
  "dependencies": {
    "express":      "3.5.x"
  }
}
```

Lastly, check your `.cfignore` file, if you use one, to make sure you don't
have an entry for `node_modules`.  If you do, change it to `/node_modules`.



caveats
================================================================================

* I don't believe the `node_modules_private` name is hard-coded anywhere, except
  in the main project's `project.json` postinstall script property.  So, feel
  free to make it shorter or whatever.  Hopefully that won't be a problem.

* You'll be uploading all the private packages, and all their dependencies, so
  your `cf push`'s will take longer than if you didn't upload any modules.  I
  don't see anyway around that, so ... deal.

* No idea if this would work with "nested" packages - eg, if your private
  package itself uses this technique to manage it's own private packages.
  Let me know!

<!-- ======================================================================= -->

[bluemix]:            https://bluemix.net
[cloud-foundry]:      http://cloudfoundry.org
[bluemix-hello-node]: https://hub.jazz.net/project/pmuellr/bluemix-hello-node/overview
[strongloop-npm]:     http://strongloop.com/strongblog/node-js-registry-mirror-rackspace/
[own-npm]:            http://stackoverflow.com/questions/14609131/can-i-run-a-private-npm-repository-without-replicating-the-public-repository
[npm-url-dep]:        https://www.npmjs.org/doc/files/package.json.html#urls-as-dependencies
[npm-scripts]:        https://www.npmjs.org/doc/misc/npm-scripts.html
[npm-link]:           https://www.npmjs.org/doc/cli/npm-link.html
[sym-links]:          http://en.wikipedia.org/wiki/Symbolic_link
