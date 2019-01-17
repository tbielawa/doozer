Will you install gh-md-toc and add a table of contents here?

https://github.com/ekalinin/github-markdown-toc




# Main workflow

The main purpose of doozer is to run brew builds based on upstream sources. This workflow consists of cloning the internal dist-git repo, cloning the upstream source repo, and rebasing the source into the dist-git with minor alterations. Both RPMs and container images are supported. We will mostly discuss image builds here.

# How to get doozer

Installing doozer is easy, just run `pip install rh-doozer` in the Python 2 environment of your choosing. We highly recommend using a virtual environment.

I tested this in a fresh `fedora:latest` container

    $ podman run -it fedora:latest '/bin/bash'
    Trying to pull docker.io/fedora:latest...Getting image source signatures
    Copying blob sha256:0be2a68855d7bbbba01b447a79c873f137e6fb47362e79f2fd79c72575c9b73a
    85.70 MB / 85.70 MB [======================================================] 6s
    Copying config sha256:26ffec5b4a8ad65083424903b7aa175953329413fe5cc4c0dac6fedbe81f2fbb
    1.99 KB / 1.99 KB [========================================================] 0s
    Writing manifest to image destination
    Storing signatures

Then I `dnf install python-pip`d. Here's what happened when I ran `pip
install --user` (`--user` so people don't require the sudo or whatever. But
that's your choice to add/not `--user`):


I had to `dnf install krb5-devel` to get past the above error.

Then I ran into an rpmdev library problem


Fixed with `dnf install python2-rpm`.

Then it couldn't build some bindings for pykerberos:


Fix with `dnf install gcc`. Which gets to another compilation failure:


Fix that by `dnf install redhat-rpm-config`.

Then we get to a problem with missing python devel packages for bindings:

Fix with `dnf install python2-devel`.

And then you're done! Simple as that!

    Successfully installed bashlex-0.12 koji-1.16.0 pyOpenSSL-18.0.0 pydotconfig-0.1.2 pygitdata-0.2.1 pykerberos-1.2.1 python-krbV-1.0.90 requests-kerberos-0.12.0 rh-doozer-0.3.13 rpm-py-installer-0.8.0


In summary:

    dnf install krb5-devel python2-rpm gcc redhat-rpm-config python2-devel

Some of those are in the readme. some are not. May as well document
all of that in one place. In this USAGE file, I'd really say "To
install doozer, follow the instructions in #installing in the README"
rather than duplicate all of that.


You will also need to install various tools used by doozer. Refer to [README.md#installation|the README].

Above link didn't actually turn into a link. Try more like how it's
down in the Dependencies below instead.

Finally, you'll need a private ssh key with access to clone sources from github.com, and probably a kerberos ticket for use with dist-git.

# Doozer setup

After installing you will need to setup your doozer config, by adding the file `~/.config/doozer/settings.yaml` with the following contents:

```
#Persistent working directory to use
working_dir: <path_to_dir>

#Git URL or File Path to build data
data_path: https://github.com/openshift/ocp-build-data.git

#Sub-group directory or branch to pull build data
group: openshift-4.0
```

May I suggest modifying the above so that it says "for red hatters,
see https://mojo.redhat.com/docs/DOC-1182562#jive_content_id_Doozer
for a doozer config file with the recommended settings.


Note, all three options above can be set at the CLI with `doozer --working-dir <path> --data-path <url> --group <group>` but we highly recommend setting them in `settings.yaml` to save typing evertime you run.
Also, please note that `group` in `settings.yaml` only makes sense if you only ever work on that one version, otherwise specify it on the CLI at runtime.

# Dependencies

You will also need to make sure that you have `imagebuilder` installed and that `docker` is properly configured to point to the internal registry. Please follow the instructions on the [Doozer Setup doc](https://github.com/openshift/doozer/blob/master/README.md#local-image-builds).

Dependencies should be grouped together with installation/getting it
stuff. Which should be all together in on space to avoid duplication
and doc rot. Like the installation part in the readme.


# Image Configuration

First, please note that you will need to have your image configured in a `doozer` compatible config repo, like [ocp-build-data](https://github.com/openshift/ocp-build-data/) as shown in the `doozer` config above. If your image is already built as part of the OCP release then it already exists in that repo and you are good to go.

If your image is not already part of the OCP build, we recommend forking `ocp-build-data` and adding it for the sake of testing. Just remember to update `settings.yaml` to point to your version.

Alternatively, you can also clone `ocp-build-data` locally and make changes without commit to it. Then specify the file path to that cloned repo instead of the URL to use that local version.
If you have questions about the layout and format of `ocp-build-data` please reach out to ART via [aos-team-art@redhat.com](mailto:aos-team-art@redhat.com)

# Running a Local Build

Once all the above is ready, you can build your image!

*Please note that yes, you need to refer to your image to `doozer` with the **distgit** name which is different than the common name for the image. In the near future ART will likely update `doozer` to take either name, but for the time being this is a historical usage quirk that's held over. So, for example, if you were to build `openshift/ose-ansible` you would need to specify `aos3-installation` which is defined by the yaml config [with the same name](https://github.com/openshift/ocp-build-data/blob/openshift-4.0/images/aos3-installation.yml) on `ocp-build-data`. All image config files in `ocp-build-data` have names that match their dist-git repo name, without exception.*

It requires 2 steps:

First, run:

Put what the command is going to do (the paragraph following) before
the command being ran. You already are doing that for most of the
examples following this one.

`doozer --local --latest-parent-version --group openshift-4.0 -i <distgit_name> images:rebase`

There are options available for `images:rebase` like the version and release strings, but they are not strictly necessary for a local build. This command will clone down your image source from the configured repo and copy the necessary files to a build directory while applying a few modifications required for the OCP build process.

Note: `--latest-parent-version` works around the assumption that doozer makes about parent-child relationships of images, which is that parent and child will be built at the same version in a single run. You will need this option almost all the time unless you are building a base image or all of the images at once.

Note that it will only pull from the configured repo and branch which is typically the released code, which for obvious reasons may be undesireable for the sake of testing. If you already have the repo with test code locally you can override that remote clone by running:

`doozer --local --latest-parent-version --group openshift-4.0 --source <distgit_name> <path_to_repo> -i <distgit_name> images:rebase`

This will then only use the local code for running the build. This operation will in no way modify the given repo, only copy the files into the `doozer` working directory.

Next, to build the image run:

`doozer --local --group openshift-4.0 -i <distgit_name> images:build`

It will now build the image using either `imagebuilder` or `docker build` as configured in the image's config yaml. The stdout of the build process will be written out to the console in realtime so that you can monitor the progress.

That's it. Once done, the image will be available to docker locally.

# Cleaning the Workspace

Please note that the given working directory (as noted above) is intended to be persistent between `doozer` steps. You can even run subequent builds of the same or other images with the same persistent working directory. However, if you switch versions of OCP you are testing against you will need to delete the contents of the directory before running `doozer`.



Overall, gg!
