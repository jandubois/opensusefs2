# opensusefs2 - an openSUSE based stack for Cloud Foundry #


## What is it? ##

`opensusefs2` is a rootfs plus buildpack support for deploying applications on openSUSE to Cloud Foundry.

It can be installed alongside the default `cflinuxfs` stack and users can decide on an app-by-app basis about which stack to use.


## Where are the bits? ##

I've forked all modified CF repos into my `jandubois` Github account and created an `opensusefs2` branch for the required modifications. The forks are all included as submodules in the [opensusefs2](https://github.com/jandubois/opensusefs2) repo.

All build artifacts have been saved to the `s3://jandubois` bucket, so that they are available to bosh releases, or for development on other machines.  The rootfs is also available from Docker Hub as [jandubois/opensusefs2](https://hub.docker.com/r/jandubois/opensusefs2/).


## How it is built? ##

During this proof-of-concept phase everything has been built manually. CF has pipelines for all the steps, but things need to work first. :)


### Building the rootfs ###

The rootfs is built using the [stacks](https://github.com/jandubois/stacks/tree/opensusefs2) repo. It uses `docker build` to create a Docker image and extracts the rootfs from it. The docker image will also be needed to build various binaries for the buildpacks.

```bash
make
docker push jandubois/opensusefs2
../bin/sha1stamp opensusefs2.tar.gz $(git describe --tags --abbrev=0)
s3cmd put opensusefs2*.tar.gz s3://jandubois
s3cmd setacl s3://jandubois --acl-public --recursive
sha1sum opensusefs2*.tar.gz
ls -l opensusefs2*.tar.gz
```

The SHA1 and file size are required to update the blob information in the next step.


### Building a bosh release for the opensusefs2 stack ###

The [opensusefs2-rootfs-release](https://github.com/jandubois/opensusefs2) repo is a fork of the [cflinuxfs2-rootfs-release](https://github.com/cloudfoundry/cflinuxfs2-rootfs-release) repo with essentially just a global search-and-replace of `cflinuxfs2` by `opensusefs2`.

**Important:** In any forked repo of a bosh release, the `.final_builds/` and `releases/` subdirectories must be deleted. Otherwise bosh may get confused with the build artifacts of upstream builds.

The only other changes are to update the bucket name in [final.yml](https://github.com/jandubois/opensusefs2-rootfs-release/blob/master/config/final.yml) to `jandubois` and to update [blob.yaml](https://github.com/jandubois/opensusefs2-rootfs-release/blob/master/config/blobs.yml) with the filename, sha, and size of the new `opensusefs2.tar.gz`.

Before pushing changes to Github test that the new blob configuration is correct by building a bosh release locally:

```bash
bosh create release --force
```

This release by itself is not yet useful, as the default buildpacks only support the `cflinuxfs2` rootfs.


### Building binary packages for the buildpacks ###

The buildpacks need additional binaries not included in the rootfs, e.g. `ruby`, `php`, `node`, `httpd`, `nginx`, etc.  These are compiled and packaged using the [binary-builder](https://github.com/jandubois/binary-builder/tree/opensusefs2).

Each buildpack includes a `manifest.yml` file that lists supported language versions, and the locations where binary builds can be downloaded. For example the Ruby [manifest.yml](https://github.com/jandubois/ruby-buildpack/blob/opensusefs2/manifest.yml) may show the usage of Ruby 2.3.3.

The `binary-builder` has a [ruby recipe](https://github.com/jandubois/binary-builder/blob/opensusefs2/recipe/ruby.rb) to download Ruby sources and package the binary into a tarball.  The `url` method shows where it downloads the sources. Since the builder needs to verify the checksum, you have to download it manually first:


```bash
curl -L -O https://cache.ruby-lang.org/pub/ruby/2.3/ruby-2.3.3.tar.gz
md5sum ruby-2.3.3.tar.gz
docker run -w /binary-builder -v `pwd`:/binary-builder -it jandubois/opensusefs2 ./bin/binary-builder --name=ruby --version=2.3.3 --md5=e485f3a55649eb24a1e2e1a40bc120df
../bin/sha1stamp ruby-2.3.3-linux-x64.tgz
s3cmd put ruby-2.3.3-linux-x64*.tgz s3://jandubois
s3cmd setacl s3://jandubois --acl-public --recursive
md5sum ruby-2.3.3-linux-x64*.tgz
ls -l ruby-2.3.3-linux-x64*.tgz
```

The MD5 hash and file size will be needed for updating the `ruby-buildpack` with this new binary.

The Ruby recipe did not require any further changes, but e.g. the PHP recipe installs a lot of packages via `apt-get` and later copies various libraries into the final tarball. All this needs to be ported to be openSUSE compatible.


### Updating buildpacks with multi-stack support ###

Each CF buildpack is maintained in it's own separate repo, e.g. Ruby in the [ruby-buildpack](https://github.com/jandubois/ruby-buildpack/tree/opensusefs2) repo. The buildpacks are configured with a `manifest.yml` that lists the versions, urls, and md5 checksums for all binary assets. The only thing required to support the `opensusefs2` stack is to add entries for all the packages built using the `binary-builder` in the previous step.

The one complication is that the `buildpack-packager` is validating the `manifest.yml` against a schema that has the official stack names [hard-coded](https://github.com/cloudfoundry/buildpack-packager/blob/4dbb1042af7585fef7fe8185a5c7aeea47695af1/lib/buildpack/packager/manifest_schema.yml#L67). For now I am just editing the schema manually after installing the gem:

```bash
$EDITOR manifest.yml
BUNDLE_GEMFILE=cf.Gemfile bundle
$EDITOR /usr/local/lib/ruby/gems/2.4.0/bundler/gems/buildpack-packager-63c7297eec83/lib/buildpack/packager/manifest_schema.yml
BUNDLE_GEMFILE=cf.Gemfile bundle exec buildpack-packager --cached
../bin/sha1stamp ruby_buildpack-cached-v1.6.34.zip
s3cmd put ruby_buildpack-cached-v1.6.34*.zip s3://jandubois
s3cmd setacl s3://jandubois --acl-public --recursive
md5sum ruby_buildpack-cached-v1.6.34*.zip
```

### Updating the java-buildpack with opensusefs2 support ###

The `java-buildpack` is maintained by the"Java Experience" team and is completely different from the other buildpacks. It doesn't rely on the `CF_STACKS` environment variable, but has it's own platform detection code (it has support for running on Redhat, and even OS X).

I've done some testing with the [java-test-applications](https://github.com/cloudfoundry/java-test-applications), and it seems like the `trusty` binaries work as-is on `opensusefs2` (commands below on OS X):

```bash
brew tap pivotal/tap
brew install springboot
brew install typesafe-activator

cd java-test-applications
./gradlew build

cd java-main-application
$EDITOR manifest.yml
cf push
```

In the `manifest.yml` you need to update the `buildpack` and `stack` keys.

So for now I've applied a simplistic [patch](https://github.com/jandubois/java-buildpack/commit/dd293d3) to the java-buildpack to pretend that any SUSE system is a `trusty` instance. This does have the undesirable side-effect that the asset URLs displayed during staging include the `trusty` string verbatim, even when building on `opensusefs2`:

```
Downloading Open Jdk JRE 1.8.0_121 from https://java-buildpack.cloudfoundry.org/openjdk/trusty/x86_64/openjdk-1.8.0_121.tar.gz (found in cache)
```

I have not used the [java-buildpack-system-test](https://github.com/cloudfoundry/java-buildpack-system-test) integration tests.


### Creating bosh releases for the multi-stack buildpacks ###

Each buildpack has a corresponding bosh release repo. For Ruby this is [ruby-buildpack-release](https://github.com/jandubois/ruby-buildpack-release/tree/opensusefs2).

These are simple forks of the upstream repos, removing `.final_buillds/` and `releases/` subdirectories, and updating the bucket name in [config/final.yml](https://github.com/jandubois/ruby-buildpack-release/blob/opensusefs2/config/final.yml) as well as the blob settings in [config/blobs.yml](https://github.com/jandubois/ruby-buildpack-release/blob/opensusefs2/config/blobs.yml).


## Adding the opensusefs2 releases to Helion Stackato ##

I've added the 3 releases (rootfs, ruby, and php buildpacks) to the (private) [hpcloud/hcf](https://github.com/hpcloud/hcf/tree/opensusefs2) repo in the `opensusefs2` branch.

The additional stack is added to the `cc.stacks` property and `cc.default_stack` is made configurable via the `DEFAULT_STACK` parameter, so a user could decide to make `opensusefs2` the default stack. The role manifest has been updated to load the multi-stacks buildpacks from their direct releases instead of loading the versions bundled with `cf-release`.

This branch builds on Jenkins and passes all tests (unsurprisingly, as they run against the default `cflinuxfs2` and ignore the additional stack).


### Deploying it on HCP ###

Jenkins has generated a set of [SDL and IDL](https://concourse-hpe.s3.amazonaws.com/hcf-hcp-1.8.8-pre%2Bcf251.35.g142117e.zip) that can be used to deploy a version of HCF with  `opensusefs2` on a working HCP setup.

I haven't created an HSM catalog entry for it, so deployment needs to be done via the REST API of HCP. First make sure that your admin user has the `publisher` role. You need to log out and back in to update the token:

```bash
hcp=https://hcp.mydomain.com:443
admin_user=admin
admin_password=secret

hcp api -k ${hcp}
hcp login --username=${admin_user} --password=${admin_password}
hcp update-user ${admin_user} --newrole=publisher
hcp logout
hcp login --username=${admin_user} --password=${admin_password}
```

Edit the `instance-basic-dev.json` IDL to set the `DOMAIN` and `cluster-admin-password`. You can also set `DEFAULT_STACK` to `opensusefs2`. Then run:

```bash
token=$(cat $HOME/.hcp | jq -r .AccessToken)
curl -k -H "Content-Type: application/json" -H "Authorization: Bearer $token" -XPOST -d @sdl.json $(hcp)/v1/services
curl -k -H "Content-Type: application/json" -H "Authorization: Bearer $token" -XPOST -d @instance-basic-dev.json $(hcp)/v1/instances
```


### Smoke and acceptance tests ###

The test suits have the `cflinuxfs2` name hard-coded in a few places, so for testing purposes I deployed the `opensusefs2.tar.gz` rootfs under the `cflinuxfs2` name.  The tests make use of Ruby and PHP buildpacks, which is the main reason they have been adapted to the `opensusefs2` stack first.

After a few iteration of fixing package names and adding additional packages that seem to be part of the `ubuntu:trusty` base image, all smoke and HCF acceptance tests (HATS) are passing.

There are 6 remaining Cloud Foundry acceptance tests (CATS) still failing. In at least [one case](https://github.com/cloudfoundry/cf-acceptance-tests/blob/master/apps/app_stack.go#L124) this is due to a hard-coded expectance of running on Ubuntu: `expected_lsb_release := "DISTRIB_CODENAME=trusty"`.


## Todo ##

This project is currently at the proof-of-concept stage.  There is a lot left to do to get feature parity / compatibility with the `cflinuxfs2` stack.  Here is a (likely incomplete) list:


### stacks ###

* Check out the errors from `zypper` during the build.

* Configure all locale settings.

* Configure timezone settings.

* `ubuntu:trusty` includes a lot of packages missing from `opensuse:leap`; compare `cflinuxfs2_dpkg_l.out` with `opensusefs2_zypper.out`.

* Figure out why the ImageMagick regression test fails on `opensusefs2`. It tests for CVEs that are supposed to be fixed in the SUSE package version.

* There are still lots of packages missing in `install_packages.sh` where the corresponding openSUSE package name wasn't obvious.

* Double check that the "obvious" packages choosen are actually correct. :)

* Figure out why there is a firstboot mechanism in `cflinuxfs2`.


### binary-builder ###

* Finish work on PHP recipe. Support for IMAP, LDAP, and SNMP is still commented out.

* Build PHP with a `php-extensions.yml` file.

* Verify all other recipes. This should probably done while porting the missing buildpacks because some problems only become apparent when running apps.

  For example `httpd` builds its own `libexpat.so.0`, but the recipe did not copy it into the package tarball because it already exists in the Ubuntu rootfs. The openSUSE rootfs only has `libexpat.so.1`.


### buildpacks ###

Only Ruby and PHP buildpacks have been ported so far.

* Build all dependencies for Ruby and PHP buildpack, not just the default version. Build jruby etc.

* Automate building of binaries for all buildpacks by processing their `manifest.yml`.

* Add support for `java_buildpack`, which is not maintained by the CF buildpack team and doesn't use `binary-builder` and `buildpack-packager`.

* The Ruby based [bosh_cli](https://github.com/cloudfoundry/bosh/tree/ruby-2.3.0/bosh_cli) is no longer in `master` and only available in the `ruby-2.3.0` branch of the bosh repo, so it seems unlikely that a change to the schema-checker will be accepted upstream.

* Check if the new [bosh-cli](https://github.com/cloudfoundry/bosh-cli) written in Go supports new stack names. Provide a patch if it doesn't.


### CATS ###

There are 6 remaining CATS failures that need to be examined.

At least one failure is due to CATS assuming Ubuntu Trusty as the stack OS. There should be an upstream patch to make these at least skippable, if not configurable to other stacks.  It should become possible to pass CATS in CI against non-default stacks.


## Production thoughts ##

Adding an `opensusefs2` stack should be completely certifiable: There are no changes in the code delivered to end-users, only additional bundled binaries. And the CF components are still being used by default.


### Ongoing maintenance ###

It seems highly unrealistic to expect the CF Foundation to start supporting an openSUSE rootfs, so this would require ongoing maintenance effort.

Changes to the `stacks`, `binary-builder` and all the particular buildpack repos would need to be monitored and ported to the corresponding forks, especially for any changes addressing CVEs.


### CI systems ###

The CF buildpacks team has a number of concourse pipelines in [buildpacks-ci](https://github.com/cloudfoundry/buildpacks-ci). Under [private repos](https://github.com/cloudfoundry/buildpacks-ci#private-repos) it lists parts of the infrastructure not (yet?) public. Looks like they are working on releasing more of it, e.g. [public-buildpacks-ci-robots](https://github.com/cloudfoundry/public-buildpacks-ci-robots). These would need to be ported and maintained to be able to test and release updates in a timely manner.


## Example ##

Here is an abbreviated transcript of pushing the same app under different names using both stacks:


```bash
$ cf stacks
name            description
cflinuxfs2      Cloud Foundry Linux-based filesystem
opensusefs2     openSUSE-based filesystem

$ cf push -s cflinuxfs2 ubuntu
...
usage: 64M x 1 instances
urls: ubuntu.hcf.jand.stacktest.io
stack: cflinuxfs2
buildpack: php 4.3.27

$ cf push -s opensusefs2 opensuse
...
usage: 64M x 1 instances
urls: opensuse.hcf.jand.stacktest.io
stack: opensusefs2
buildpack: php 4.3.27

$ cf ssh ubuntu -c "cat /etc/os-release"
NAME="Ubuntu"
VERSION="14.04.5 LTS, Trusty Tahr"
ID=ubuntu
ID_LIKE=debian
PRETTY_NAME="Ubuntu 14.04.5 LTS"
VERSION_ID="14.04"
HOME_URL="http://www.ubuntu.com/"
SUPPORT_URL="http://help.ubuntu.com/"
BUG_REPORT_URL="http://bugs.launchpad.net/ubuntu/"

$ cf ssh opensuse -c "cat /etc/os-release"
NAME="openSUSE Leap"
VERSION="42.2"
ID=opensuse
ID_LIKE="suse"
VERSION_ID="42.2"
PRETTY_NAME="openSUSE Leap 42.2"
ANSI_COLOR="0;32"
CPE_NAME="cpe:/o:opensuse:leap:42.2"
BUG_REPORT_URL="https://bugs.opensuse.org"
HOME_URL="https://www.opensuse.org/"
```
