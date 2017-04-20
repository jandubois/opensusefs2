# Deploying CF v255 with opensusefs2 on bosh-lite #

This README shows how to install CF v255 with just the `opensusefs2` stack; it will not install the `cflinuxfs2` stack at all.

TL;DR: Watch the brief (3:45m) [asciicast](https://asciinema.org/a/17xp6rkgu728hszi6r4yu1d63), based on running the commands in this [script](./asciinema.txt).

## Set up bosh-lite vagrant box ##

Install Vagrant and Virtualbox:

```bash
sudo zypper in -y virtualbox vagrant
```

I installed the `bosh` client directly with the system Ruby. I would be better to follow the [dev-workstation setup](https://github.com/SUSE/suse-paas-planning/blob/master/docs/dev-workstation.md#ruby) instructions and use a user-installed version instead.

```bash
sudo gem install bosh_cli --no-ri --no-rdoc
alias bosh=bosh.ruby2.1
```

Follow the [bosh-lite instructions](https://github.com/cloudfoundry/bosh-lite#install-bosh-lite) on setting up a running instance; I needed to use at least 14GB for the VM to be able to finish CATS without spurious errors.

```bash
cd ~/workspace/bosh-lite
VM_MEMORY=14336 vagrant up
bin/add-route
bosh target 192.168.50.4 lite
```

## Upload and deploy CF v255 with opensusefs2 ##

```bash
bosh upload stemcell https://bosh.io/d/stemcells/bosh-warden-boshlite-ubuntu-trusty-go_agent
bosh upload release https://bosh.io/d/github.com/cloudfoundry/cf-release?v=255
bosh upload release https://bosh.io/d/github.com/cloudfoundry/diego-release?v=1.11.0
bosh upload release https://bosh.io/d/github.com/cloudfoundry/garden-runc-release?v=1.3.0
bosh upload release https://s3.amazonaws.com/cf-opensusefs2/opensusefs2-0%2Bdev.2.tgz
```

The `opensusefs2-release` contains both the rootfs and the updated buildpacks. Note that this particular release is still based on the assets created during Hackweek: the buildpack versions match the ones from CF v251, and they only include openSUSE assets for the default version of their respective runtimes.

The `cf.yml` and `diego.yml` deployment manifests in this directory are based on the ones generated by the scripts for bosh-lite, but [changed](https://github.com/jandubois/opensusefs2/commit/7cb9c14) to remove the `dotnet-core-buildpack` and switch the default backend to `diego`, and then further [modified](https://github.com/jandubois/opensusefs2/commit/1e8b13c) to use the `opensusefs2` stack exclusively.

```bash
bosh -n -d cf.yml deploy
bosh -n -d diego.yml deploy
```

## Install cf cli ##

```bash
sudo zypper ar -r http://download.opensuse.org/repositories/Cloud:/Tools/openSUSE_Leap_42.2/Cloud:Tools.repo
sudo zypper in cf-cli
```

## Login and setup org and space ##

Also enable the `diego_docker` feature flag; it is required to run the acceptance tests later.

```bash
cf login -a api.bosh-lite.com -u admin -p admin --skip-ssl-validation
cf enable-feature-flag diego_docker

cf create-org suse
cf target -o suse
cf create-space myspace
cf target -o suse -s myspace
```

## Deploy the go-env sample app

```bash
git clone git@github.com:stackato-samples/go-env.git
cd go-env

# Remove the hardcoded `stack` and `buildpack` lines
vim manifest.yml

cf push
curl -s go-env.bosh-lite.com | sort

# Verify that the app is running on the openSUSE stack
cf ssh go-env -c "cat /etc/os-release" | egrep -i "suse|"

cf delete -f go-env
```

## Run smoke tests ##

```bash
sudo zypper in -y go mercurial bzr
mkdir $HOME/go
export GOPATH=$HOME/go

go get -d github.com/cloudfoundry/cf-smoke-tests
cd ~/go/src/github.com/cloudfoundry/cf-smoke-tests/
export CONFIG=$HOME/smoke.json
./bin/test
```

## Run acceptance tests ##

I found it very hard to get CATS to pass on bosh-lite until I increased the VM memory size to 14GB.  Discussion on [#capi](https://cloudfoundry.slack.com/archives/C07C04W4Q/p1491845284341164) and [#release-integration](https://cloudfoundry.slack.com/archives/C0FAEKGUQ/p1491953758062567) on the CF Slack showed that bosh-lite is generally considered "flakey" by the upstream people.

The test suites selected in the [config file](./cats.json) are generally the same as what we used for HCF tests on Jenkins. I've doubled all timeout values.

One of the acceptance tests has a hardcoded dependency on Ubuntu Trusty. The `opensusefs2` branch in my fork changes it to a dependency on `opensusefs2`.

```bash
go get -d github.com/cloudfoundry/cf-acceptance-tests
cd ~/go/src/github.com/cloudfoundry/cf-acceptance-tests

git remote add jandubois git@github.com:jandubois/cf-acceptance-tests.git
git fetch jandubois
git merge --commit --no-edit jandubois/opensusefs2

export CONFIG=$HOME/cats.json
./bin/test -r -slowSpecThreshold=120 -skipPackage=helpers -keepGoing -nodes=1
```