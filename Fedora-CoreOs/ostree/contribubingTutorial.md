# OSTree Contributing Tutorial

The following guide is about OSTree forking, building, adding a command, testing the command, and submitting the change.

    Getting Started
    Building OSTree
        Install Build Dependencies
        OSTree Build Commands
            Notes
            Tip
    Testing a Build
        Testing in a Container
        Testing in a Virtual Machine
    Tutorial: Adding a basic builtin command to ostree
        Modifying OSTree
        OSTree Tests
        Submitting a Patch
        Returning Workflow

## Getting Started

Fork https://github.com/ostreedev/ostree, then run the following commands.
```
$ git clone https://github.com/<username>/ostree && cd ostree
$ git remote add upstream https://github.com/ostreedev/ostree
$ git checkout master
$ git fetch upstream && git branch --set-upstream-to=upstream/master master
```

Make a branch from master for your patch.
```
$ git checkout -b <name-of-branch>
$ git branch --set-upstream-to=upstream/master <name-of-branch>
```

## Building OSTree

## Install Build Dependencies

Execute one of the following group commands as superuser depending on your machine’s package manager.

For Fedora:
```
$ dnf install @buildsys-build dnf-plugins-core && \
dnf builddep ostree
```

For CentOS:
```
$ yum install yum-utils dnf-plugins-core && \
yum-builddep ostree
```

For Debian based distros:
```
$ apt-get update && \
apt-get install build-essential && \
apt-get build-dep ostree
```

build.sh will have a list of packages needed to build ostree.

## OSTree Build Commands

These are the basic commands to build OSTree. Depending on the OS that OSTree will be built for, the flags or options for ./autogen.sh and ./configure will vary.

See ostree-build.sh in this tutorial below for specific commands to building OSTree for Fedora 28 and Fedora 28 Atomic Host.
```
# optional: autogen.sh will run this if necessary
git submodule update --init

env NOCONFIGURE=1 ./autogen.sh

# run ./configure if makefile does not exist
./configure

make
make install DESTDIR=/path/to/install/binary
```

**Notes**

Running git submodule update --init is optional since autogen.sh will check to see if one of the submodule files for example from libglnx/ or from bsdiff/ exists.

Additionally, autogen.sh will check to see if the environment variable NOCONFIGURE is set. To run ./configure manually, run autogen in a modified environment as such, env NOCONFIGURE=1 ./autogen.sh.

Otherwise, leave NOCONFIGURE empty and autogen.sh will run ./configure as part of the autogen.sh command when it executes.

For more information on --prefix see Variables for Installation Directories.

make install will generate files for /bin and /lib. If DESTDIR is unspecified then OSTree will be installed in the default directory i.e. /usr/local/bin and its static libraries in /usr/local/lib. Note that the /usr/local portion of the path can be changed using the --prefix option for ./configure.

See this GNU guide on DESTDIR Staged Installs for more information.
Tip

Make allows parallel execution of recipes. Use make -j<N> to speed up the build. <N> is typically $((2 * $(nproc))) for optimal performance, where nproc is the number of processing units (CPU cores) available.

See page 106 of the GNU Make Manual for more information about the --jobs or -j option.

## Testing a Build

It is best practice to build software (definitely including ostree) in a container or virtual machine first.

### Testing in a Container

There are a variety of container engines available and many distributions have pre-packaged versions of e.g. Podman and Docker.

If you choose to use Docker upstream, you may want to follow this post-installation guide for Docker. This will allow you to run Docker as a non-root user on a Linux based host machine.

You will need to have pushed a remote git branch $REMOTE_BRANCH (see ostree-git.sh below) in order to pull your changes into a container.

The example below uses Docker to manage containers. Save the contents of this Dockerfile somewhere on your machine:
```
# this pulls the fedora 28 image
FROM registry.fedoraproject.org/fedora:28

# install ostree dependencies
RUN dnf update -y && \
    dnf -y install @buildsys-build dnf-plugins-core  && \
    dnf -y builddep ostree  && \
    dnf clean all

# clone ostree and update master branch
COPY ostree-git.sh /
RUN ../ostree-git.sh

# builds ostree + any additional commands
COPY ostree-build.sh /

# entry into the container will start at this directory
WORKDIR /ostree

# run the following as `/bin/sh -c`
# or enter the container to execute ./ostree-build.sh
RUN ../ostree-build.sh
```

Save the following bash scripts in the same directory as the Dockerfile. Then change the mode bit of these files so that they are executable, by running chmod +x ostree-git.sh ostree-build.sh
```
#!/bin/bash

# ostree-git.sh
# Clone ostree and update master branch

set -euo pipefail

# Set $USERNAME to your GitHub username here.
USERNAME=""

# clone your fork of the OSTree repo, this will be in the "/" directory
git clone https://github.com/$USERNAME/ostree.git
cd ostree

# Add upstream as remote and update master branch
git checkout master  
git remote add upstream https://github.com/ostreedev/ostree.git
git pull --rebase upstream master
```
```
#!/bin/bash

# ostree-build.sh
# Build and test OSTree

set -euo pipefail

# $REMOTE_BRANCH is the name of the remote branch in your
# repository that contains changes (e.g. my-patch).
REMOTE_BRANCH=""

# fetch updates from origin
# origin url should be your forked ostree repository
git fetch origin

# go to branch with changes
# if this branch already exists then checkout that branch
exit_code="$(git checkout --track origin/$REMOTE_BRANCH; echo $?)"
if [[ "$exit_code" == 1 ]]
then
    echo "This branch:" $REMOTE_BRANCH "is not a remote branch."
    exit
fi

# make sure branch with changes is up-to-date
git pull origin $REMOTE_BRANCH

# build OSTree commands for Fedora 28 and Fedora 28 Atomic Host
./autogen.sh --prefix=/usr --libdir=/usr/lib64 --sysconfdir=/etc
./configure --prefix=/usr
make -j$((2 * $(nproc)))
make install

# any additional commands go here
```

### Build the container

Run docker build in the same directory of the Dockerfile like so:
```
$ docker build -t ostree-fedora-test .
```

When this build is done, the -t option tags the image as ostree-fedora-test.

**Note:** Do not forget the dot . at the end of the above command which specifies the location of the Dockerfile.

You will see ostree-fedora-test listed when running docker images:
```
REPOSITORY                                 TAG                 IMAGE ID            CREATED             SIZE
ostree-fedora-test                         latest              817c04cc3656        1 day ago          978MB
```

### Entering the Container

To start the ostree-fedora-test container, run:

$ docker run -it --rm --entrypoint /bin/sh --name ostree-testing ostree-fedora-test

**Note:**

--rm option tells Docker to automatically clean up the container and remove the file system when the container exits. Otherwise remove it manually by running docker rm <container name>.

The state of the container will not be saved when the shell prompt exits. Best practice is modify the Dockerfile to modify the image.

### Testing in a Container Workflow

    Edit the changes to OSTree on your local machine.
    git add to stage the changed files, git commit and then git push origin <local-branch>:<remote-branch>.

    Testing on a new container vs. Testing on an existing container:

    If the ostree-testing container was newly built right after your changes have been committed, then the container’s build of ostree should contain your changes.

    Else: Within the ostree-testing container, run ../ostree-build.sh in the ostree directory. This will pull in changes from your branch and create a new ostree build.

    make install will install OSTree in the default location i.e. /usr/..in a Fedora 28 container.
    Test ostree.

## Testing in a Virtual Machine

To create a Fedora 28 Atomic Host Vagrant VM, run the following commands:
```
$ mkdir atomic && cd atomic
$ vagrant init fedora/28-atomic-host && vagrant up
```

An option is to use rsync to transfer ostree files to a Vagrant VM.

To find the IP address of a Vagrant VM, run vagrant ssh-config in the same directory as the Vagrantfile.

Steps to rsync files to test an ostree build:

    Copy the contents of your public ssh key on your host machine e.g. id_rsa.pub to /home/vagrant/.ssh/authorized_keys on the VM.

    Run sudo su, followed by ssh localhost then press Ctrl+c to exit from the decision prompt. This will create the .ssh directory with the right permissions.

    Using Vagrant as the user, run sudo cp ~/.ssh/authorized_keys /root/.ssh/. So that user root has the the same login credentials.

    To override the Read-only file system warning, run sudo ostree admin unlock.

    <ostree-install-dir> will serve as the local install location for ostree and the path to this directory should be absolute when specified in DESTDIR.

    Set rsync to sync changes in /etc and /usr from <ostree-install-dir>/ on the host to the VM:
```
     $ rsync -av <ostree-install-dir>/etc/ root@<ip-address>:/etc
     $ rsync -av <ostree-install-dir>/usr/ root@<ip-address>:/usr
```

    Using option -n will execute the commands as a trial, which is helpful to list the files that will be synced.

    Run the commands in step 6 each time a new ostree build is executed to update the change. Running ls -lt in the directory where the changed file is expected, is a simple way to check when a particular file was last modified. Proceed to the test changes ostree with the most recent changes.

## Tutorial: Adding a basic builtin command to ostree

### Modifying OSTree

This will add a command which prints Hello OSTree! when ostree hello-ostree is entered.

    Create a file in src/ostree named ot-builtin-hello-ostree.c. Code that lives in here belongs to OSTree, and uses functionality from libostree.

    Add the following to ot-builtin-hello-ostree.c:
```
     #include "config.h"

     #include "ot-main.h"
     #include "ot-builtins.h"
     #include "ostree.h"
     #include "otutil.h"

     // Structure for options such as ostree hello-ostree --option.
     static GOptionEntry options[] = {
       { NULL },
     };

     gboolean
     ostree_builtin_hello_ostree (int argc, char **argv, OstreeCommandInvocation *invocation, GCancellable *cancellable, GError **error)
     {
       g_autoptr(GOptionContext) context = NULL;
       g_autoptr(OstreeRepo) repo = NULL;
       gboolean ret = FALSE;

       // Creates new command context, ready to be parsed.
       // Input to g_option_context_new shows when running ostree <command> --help
       context = g_option_context_new ("");

       // Parses the command context according to the ostree CLI.
       if (!ostree_option_context_parse (context, options, &argc, &argv, invocation, &repo, cancellable, error))
         goto out;

       g_print("Hello OSTree!\n");

       ret = TRUE;
      out:
       return ret;
     }
```

    This defines the functionality for hello-ostree. Now we have to make sure the CLI can refer to the execution function, and that autotools knows to build this file.

    Add the following in src/ostree/main.c:
```
     { "hello-ostree",               // The name of the command
       OSTREE_BUILTIN_FLAG_NO_REPO,  // Flag not to require the `--repo` argument, see "ot-main.h"
       ostree_builtin_hello_ostree,  // Execution function for the command
       "Print hello message" },      // Short description to appear when `ostree hello-ostree --help` is entered
```

    Add a macro for the function declaration of ostree_builtin_hello_ostree, in ot-builtins.h:
```
     BUILTINPROTO(hello_ostree);
```

    This makes the function definition visible to the CLI.

    Configure automake to include ot-builtin-hello-ostree.c in the build, by adding an entry in Makefile-ostree.am under ostree_SOURCES:
```
     src/ostree/ot-builtin-hello-ostree.c \
```

    Rebuild ostree:
```
     $ make && make install DESTDIR=/path/to/install/the/content
```

    Execute the new ostree binary, from where you installed it:
```
     $ ostree hello-ostree
     Hello OSTree!
```

## OSTree Tests

Tests for OSTree are done by shell scripting, by running OSTree commands and examining output. These steps will go through adding a test for hello-ostree.

    Create a file in tests called test-hello-ostree.sh.

    Add the following to test-hello-ostree.sh:
```
     set -euo pipefail           # Ensure the test will not silently fail

     . $(dirname $0)/libtest.sh  # Make libtest.sh functions available

     echo "1..1"                 # Declare which test is being run out of how many

     pushd ${test_tmpdir}

     ${CMD_PREFIX} ostree hello-ostree > hello-output.txt
     assert_file_has_content hello-output.txt "Hello OSTree!"

     popd

     echo "ok hello ostree"      # Indicate test success
```

    Many tests require a fake repository setting up (as most OSTree commands require --repo to be specified). See test-pull-depth.sh for an example of this setup.

    Configure automake to include test-hello-ostree.sh in the build, by adding an entry in Makefile-tests.am under _installed_or_uninstalled_test_scripts:
```
     tests/test-hello-ostree.sh \
```
    Make sure test-hello-ostree.sh has executable permissions!
```
     $ chmod +x tests/test-hello-ostree.sh
```

    Run the test:
```
     $ make check TESTS="tests/test-hello-ostree.sh"
```

    Multiple tests can be specified: make check TESTS="test1 test2 ...". To run all tests, use make check.

    Hopefully, the test passes! The following will be printed:
```
     PASS: tests/test-hello-ostree.sh 1 hello ostree
     ============================================================================
     Testsuite summary for libostree 2018.8
     ============================================================================
     # TOTAL: 1
     # PASS:  1
     # SKIP:  0
     # XFAIL: 0
     # FAIL:  0
     # XPASS: 0
     # ERROR: 0
     ============================================================================
```

## Submitting a Patch

After you have committed your changes and tested, you are ready to submit your patch!

You should make sure your commits are placed on top of the latest changes from upstream/master:
```
$ git pull --rebase upstream master
```

To submit your patch, open a pull request from your forked repository. Most often, you’ll be merging into ostree:master from <username>:<branch name>.

If some of your changes are complete and you would like feedback, you may also open a pull request that has WIP (Work In Progress) in the title.

Before a pull request is considered merge ready, your commit messages should fall within the specified guideline. See Commit message style.

See CONTRIBUTING.md for information on squashing commits, and alternative options to submit patches.

## Returning Workflow

When returning to work on a patch, it is recommended to update your fork with the latest changes in the upstream master branch.

If creating a new branch:
```
$ git checkout master
$ git pull upstream master
$ git checkout -b <name-of-patch>
```

If continuing on a branch already created:
```
$ git checkout <name-of-patch>
$ git pull --rebase upstream master
```
