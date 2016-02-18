# Developing OpenStack Charms

## Overview

The OpenStack charms are part of the OpenStack project, and follow the same development process as other projects.

For full details please refer to the OpenStack [development documentation][].  This also includes details on signing up to become an OpenStack foundation member and getting setup with Gerrit and Launchpad to start development.

[development documentation]: http://docs.openstack.org/infra/manual/developers.html

## Git repositories

### Naming

All OpenStack charm repositories can be found on [github.com][] under the /openstack organisation.

Repositories are named `charm-<charm-name>` in-order to namespace the repositories within the /openstack organization appropriately.

[github.com]: https://github.com/openstack?query=charm

### Branches

All charm repositories will contain two branches; the master branch contains the current development focus, and the stable branch contains the current stable released charm.

### Tags

The OpenStack charm set produces an integrated release of charms every 3 months; these are tagged within the repository with YY.MM, for example 16.01 being the charm release from January 2016.  The most recent of these will also form the foundation of the stable branch.

## Worflow

Broadly the workflow for making a change to a charm is:

```
git clone http://github.com/openstack/charm-cinder
cd charm-cinder
git checkout -b bug/XXXXXX master
```

Make the changes you need within the charm, and then run unit and pep8 tests using tox:

```
tox
```

resolve any problems this throws up, and then commit your changes:

```
git add .
git commit
```

Commit messages should have an appropriate title, and more detail in the body; they can also refer to bugs:

```
Closes-Bug: #######
Partial-Bug: #######
Related-Bug: #######
```

Gerrit will automatically link your proposal to the bug reports on launchpad and mark them fix commited when changes are merged.

Finally submit your change for review:

```
git review
```

This will push your proposed changes to Gerrit and provide you with a URL for the review board on [review.openstack.org][]

To make amendments to your proposed change, update your local branch and then:

```
git commit --amend
git review
```

[review.openstack.org]: http://review.openstack.org


Read the OpenStack [development documentation][] for full details.

## Stable charm updates

Any update to a stable charm must first be applied into the master branch; it should then be cherry-picked in a review for the stable branch:

```
git checkout -b bug/XXXXX-stable stable
git cherry-pick -x <hash of master branch commit>
git review
```

# Charm Testing

Every proposed change to a charm is run through the following tests during the review 'check' process:

* Merge check
* pep8/lint checks (including charm proof)
* unit tests

in addition a subset of amulet tests will be executed (specified within the charm).

Once a reviewer has +2'ed as proposed change, the same checks are re-executed and the full set of amulet tests are completed.

Only once the 'gate' process has completed successfully will the change be landed in the target repository and branch.
