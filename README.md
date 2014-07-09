bluemix-private-packages
================================================================================

This project shows how to use "private" node packages in your Bluemix app.

The project is based on the super-simplistic [bluemix-hello-node][] sample
application; check that app if you're curious about the basics of building
a node app for Bluemix.



what is a "private" package?
================================================================================

For purposes of this example, a "private" package is a node package not stored
at the package repository used when you run `npm install`.  By default, that's
the one at <http://npmjs.org>, but
[you can use a different one][strongloop-npm].

Why would you use a private package in the first place?  Perhaps you have
packages which you don't want to make public ever, or at least yet.  Or maybe
the packages aren't neccessarily things you need to keep private, but are things
that aren't generally useful for other people, and so there's no real need to
put the packages in a public npm repository.



where are "private" packages stored?
================================================================================

So, where exactly **IS** a private package stored?  

You can run [your own version of npm][own-npm],
but that's hard, and there is an easier answer.  The trick is that the
`dependencies` property of the `package.json` file can contain, besides
the usual version spec, [a URL to a tarball or URL to a git repo][npm-url-dep].

Using a git url form is nice, as it's supports `commit-ish` fragments, so you
can use tags, sha's and branch names, rather than just pulling from head.

In the sample app, what we'll be showing is referencing a dependency of an
application as a git url - in this specific case, it's pointing to a git url
at GitHub, but this story should work fine for git repos you might have behind
your network's firewall, or even on your own machine.

To get started with the sample app provided in this project,
open a terminal session on your development box, and run:

    git clone https://github.com/pmuellr/bluemix-private-packages.git

And then

    cd bluemix-private-packages



general work flow
================================================================================

What we'll be doing is keeping our private packages in a new subdirectory,
`node_modules_private`.  That directory has a `package.json` file, with the
dependencies for our private packages.  The contents for this example are:

```js
{
  "name": "node_modules_private",
  "version": "0.0.0",
  "dependencies": {
    "cfenv": "git://github.com/cloudfoundry-community/node-cfenv.git#0.2.0"
  }
}
```

The URL for the `cfenv` package points to a GitHub repo, and the `commit-ish`
fragment identifies a particular tag.

You'll need to run `npm install` in this directory, every time you need
to update the dependencies.  Hopefully, this isn't that often.

Go ahead and try it now:

    cd node_modules_private
    npm install

You should see the following output:

    npm WARN package.json node_modules_private@0.0.0 No description
    npm WARN package.json node_modules_private@0.0.0 No repository field.
    npm WARN package.json node_modules_private@0.0.0 No README data
    npm http GET https://registry.npmjs.org/js-yaml
    ...
    cfenv@0.2.0 node_modules/cfenv
    +-- ports@1.1.0
    +-- underscore@1.6.0
    +-- js-yaml@3.0.2 (esprima@1.0.4, argparse@0.1.15)

Note that this `npm install` will **ONLY** be run on your development
machine, so you could do something besides `npm install` to populate
this directory with your private packages.  Pull them from anywhere.

All done in this directory, so let's go back to the main directory:

    cd ..

You should now be back in the `bluemix-private-packages` directory again.

You can now run `npm install`, to install all the public packages.
Besides the usual output, you should see some additional output like this:

    ...

    > bluemix-private-packages@ postinstall /Users/pmuellr/Projects/bluemix/bluemix-private-packages
    > sh install-private-packages.sh

    copying private module cfenv
    ...

That output comes from a [postinstall script][npm-scripts].
Specically, here's the `package.json` for the main app:

    {
      "name":           "bluemix-private-packages",
      "scripts": {
        "start":        "node server",
        "postinstall":  "sh install-private-packages.sh"
      },
      "dependencies": {
        "express":      "3.5.x"
      }
    }

The postinstall script we've specified will run the shell script
`install-private-packages.sh`.  Here's what it does:

    # ensure we're in same directory as this script
    cd `dirname $0`

    #-------------------------------------------------------------------------------
    echo "copying private module cfenv"
    cp -R node_modules_private/node_modules/cfenv node_modules

It copies the files we previously `npm install`d from within `node_modules_private`,
into the `node_modules` directory itself.  So now those packages are available
to our main program.



uploading private packages, but not saving in git
================================================================================

In my own projects, I always put `node_modules` in my `.gitignore` file, so that
the packages I use aren't stored in git.  And I also arrange to not upload my
`node_modules` directory when I run `cf push`, by adding `node_modules` to my
`.cfignore` file.  So my `.cfignore` file usually looks like this:

    node_modules
    tmp

However, there's a problem with this.  This will cause the `node_modules` directory
in `node_modules_private` to **ALSO** be ignored, which isn't what we want,
because we need those files to be uploaded to Bluemix when you push.

Luckily, the fix here is easy, as you can prepend a `/` to the `node_modules`
entry in your `.cfignore` file, and it will only ignore the top-level
`node_modules` directory.  Here's the `.cfignore` file contents for this app:

    /node_modules
    tmp



running the app
================================================================================

If you've run `npm install` in the `node_modules_private` directory, and
then run `npm install` from the main app directory, you should be able to
run the app with:

    npm start

To run the app on Bluemix, you can push it with the command

    cf push

Look for the following lines, which show copying your private packages,
which you've uploaded from your computer, into the app's `node_modules`
directory.

    > bluemix-private-packages@ postinstall /tmp/staged/app
    > sh install-private-packages.sh
    copying private module cfenv

And that's it!



<!-- ======================================================================= -->

[bluemix-hello-node]: https://hub.jazz.net/project/pmuellr/bluemix-hello-node/overview
[strongloop-npm]:     http://strongloop.com/strongblog/node-js-registry-mirror-rackspace/
[own-npm]:            http://stackoverflow.com/questions/14609131/can-i-run-a-private-npm-repository-without-replicating-the-public-repository
[npm-url-dep]:        https://www.npmjs.org/doc/files/package.json.html#urls-as-dependencies
[npm-scripts]:        https://www.npmjs.org/doc/misc/npm-scripts.html
