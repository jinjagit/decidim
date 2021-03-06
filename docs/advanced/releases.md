# Releasing new versions

In order to release new version you need to be owner of all the gems at RubyGems, ask one of the owners to add you before releasing. (Try `gem owner decidim`).

Remember to follow the Gitflow branching workflow.

## Create the stable branch for the release

1. Go to develop with `git checkout develop`
1. Create the release branch `git checkout -b release/x.y-stable && git push origin release/x.y-stable`.
1. If required, add the release branch to Crowdin so that any pending translations will generate a PR to this branch.

Mark `develop` as the reference to the next release:

1. Go back to develop with `git checkout develop`
1. Turn develop into the `dev` branch for the next release:
    1. Update `.decidim-version` to the new `dev` development version: `x.y.z.dev`, where `x.y.z` is the new semver number for the next version
    1. Run `bin/rake update_versions`, this will update all references to the new version.
    1. Run `bin/rake bundle`, this will update all the `Gemfile.lock` files
    1. Run `bin/rake webpack`, this will update the JavaScript bundle.
1. Update `SECURITY.md` and change the supported version to the new version.
1. Update the `CHANGELOG.MD`. At the top you should have an `Unreleased` header with the `Added`, `Changed`, `Fixed` and `Removed` empty sections. After that, the header with the current version and link with the same beforementioned sections and a `Previous versions` header at the bottom that links to the previous minor release stable branch.

  ```markdown
  ## [Unreleased](https://github.com/decidim/decidim/tree/HEAD)

  ### Added

  ### Changed

  ### Fixed

  ### Removed

  ## [v0.XX.0](https://github.com/decidim/decidim/releases/tag/v0.XX.0)

  ### Added
  ...

  ## Previous versions

  Please check [0.XX-stable](https://github.com/decidim/decidim/blob/release/0.XX-stable/CHANGELOG.md) for previous changes.
  ```

1. Push the changes `git add . && git commit -m "Bump develop to next release version" && git push origin develop`

## Producing the CHANGELOG.md

Look for the "Bump develop to next release version" commit sha1. You can do it either visually with `gitk .decidim-version` or by blaming `git blame .decidim-version`.

Here you have different options to see what happened from one revision to another:

```bash
git log v0.20.0..v0.20.1 --grep " (#[0-9]\+)" --oneline
git log 7579b34f55e4dcfff43c160edbbf14a32bf643b2..HEAD --grep " (#[0-9]\+)" --oneline
git log 7579b34f55e4dcfff43c160edbbf14a32bf643b2..HEAD --grep " (#[0-9]\+)"
git log 7579b34f55e4dcfff43c160edbbf14a32bf643b2..HEAD --pretty=format:"commit %h%n%n%s%n%nNotes:%n%n%N"  > full_log
```

That last command is the one used to generate log content for the GitLogParser. The `GitLogParser` translates git-log entries into our `CHANGELOG.mv` entries.
It also groups the commits if they have a git note with the format: `{groupname}: {affected_modules}`, for example `Fixed: **decidim-core**, **decidim-proposals**`. This last example will add the commit to the "Fixed" group and prefix the commit message with "- **decidim-core**, **decidim-proposals**".
The number at the end of each commit message is also replaced for a markdown link to the commit.

There's how to run the parser rake task:

```bash
# remember to have the latest commits and notes locally
git pull && git fetch origin refs/notes/*:refs/notes/*
# generate the git log in the expected format
git log 7579b34f55e4dcfff43c160edbbf14a32bf643b2..HEAD --pretty=format:"commit %h%n%n%s%n%nNotes:%n%n%N"  > full_log
# parse the git log in the expected format and produce grouped changelog entries
bin/rake parse_git_log[full_log]
```

You can now copy+paste the output of this task to each of groups in the `CHANGELOG.md`.

## Release Candidates

Release Candidates are the same as beta versions. They should be ready to go to production, but publicly released just before in order to be widely tested.

If this is a **Release Candidate version** release, the steps to follow are:

1. Checkout the release stable branch `git checkout release/x.y-stable`.
1. Update `.decidim-version` to the new version `x.y.z.rc1`
1. Run `bin/rake update_versions`, this will update all references to the new version.
1. Run `bin/rake bundle`, this will update all the `Gemfile.lock` files
1. Run `bin/rake webpack`, this will update the JavaScript bundle.
1. Commit all the changes: `git add . && git commit -m "Bump to rcXX version" && git push origin release/x.y-stable`.
1. Tag the release candidate with `git tag vX.Y.Z.rcN && git push origin vX.Y.Z.rcN`.
1. Usually, at this point, the release branch is deployed to meta-decidim during, at least, one week to validate the stability of the version.

### During the validation period

1. During the validation period, bugfixes must be implemented directly to the current `release/x.y.z-stable` branch and ported to `develop`.
1. During the validation period, translations to the officially supported languages must be added to Crowdin and, when completed, merged into `release/x.y.z-stable`.

## Major/Minor versions

Release Candidates will be tested in a production server (usually meta-decidim) during some period of time (a week at least), when they are considered ready, it is time for them to be merged into `master`:

1. Checkout the release stable branch `git checkout release/x.y-stable`.
1. Update `.decidim-version` by removing the `.rcN` suffix, leaving a clean version number like `x.y.z`
1. Run `bin/rake update_versions`, this will update all references to the new version.
1. Run `bin/rake bundle`, this will update all the `Gemfile.lock` files
1. Run `bin/rake webpack`, this will update the JavaScript bundle.
1. Commit all the changes: `git add . && git commit -m "Bump to v0.XX.0 final version" && git push origin release/x.y-stable`.
1. Create the PR for the new version.
    1. `git checkout master && git checkout -b release/x.y.z`
    1. `git merge release/x.y-stable`
    1. `git checkout --theirs *`
    1. `git checkout --theirs .github/* \.*`
    1. Review changes in `CHANGELOG.md`, manually update and create a "changelog" commit if required.
    1. `git push origin release/x.y.z`
    1. Create the PR. The base for this PR should be `master`, but GitHub may crash if there are a lot of changes. As a workaround create the branch against `develop` and, when created, change the base to `master`.
    1. Still don't merge it.
1. Before merging the PR to upgrade `master`, check that the stable branch for the previous version exists. For instance, if we are going to release v0.22.0, there should be a `release/0.21-stable` branch in the repository. If such branch does not exists, it has to be created now, before merging the new release. So, if this is the release of v0.22.0, branch off `release/0.21-stable` from `master`. These stable branches will be able to receive bugfixes, backports and will be the origin of patch releases for older releases.
1. Merge (after proper peer review) the PR to `master` and remove `release/x.y.z` branch.
1. Run `git checkout master && bin/rake release_all`, this will create all the tags, push the commits and tags and release the gems to RubyGems.
1. Once all the gems are published you should create a new release at this repository, just go to the [releases page](https://github.com/decidim/decidim/releases) and create a new one.
1. Create the stable branch for the current version. From `master`: `git checkout -b release/x.y-stable && git push origin release/x.y-stable`.
1. Update Decidim's Docker repository as explained in the Docker images section below.
1. Update Crowdin synchronization configuration with Github:
    1. Add the new `release/x.y-stable` branch.
    1. Remove from Crowdin branches that are not officially supported anymore. That way they don't synchronize with Github.
1. Update the `CHANGELOG.MD` in `release/x.y-stable`. At the top you should have an `Unreleased` header with the `Added`, `Changed`, `Fixed` and `Removed` empty sections. After that, the header with the current version. Add the `Unreleased` section or create the new current version section.

## Releasing patch versions

Releasing new versions from a ***release/x.y-stable*** branch is quite easy. The process is very similar from releasing a new Decidim version:

1. Checkout the branch you want to release: `git checkout -b release/x.y-stable`
1. Update `.decidim-version` to the new version number.
1. Run `bin/rake update_versions`, this will update all references to the new version.
1. Run `bin/rake bundle`, this will update all the `Gemfile.lock` files
1. Run `bin/rake webpack`, this will update the JavaScript bundle.
1. Update `CHANGELOG.MD`. At the top you should have an `Unreleased` header with the `Added`, `Changed`, `Fixed` and `Removed` empty sections. After that, the header with the current version and link like `## [0.20.0](https://github.com/decidim/decidim/tree/v0.20.0)` and again the headers for the `Added`, `Changed`, `Fixed` and `Removed` sections.
1. Commit all the changes: `git add . && git commit -m "Prepare VERSION release"`
1. Run `bin/rake release_all`, this will create all the tags, push the commits and tags and release the gems to RubyGems.
1. Once all the gems are published you should create a new release at this repository, just go to the [releases page](https://github.com/decidim/decidim/releases) and create a new one.
1. Update Decidim's Docker repository as explained in the Docker images section.

## Docker images for each release

1. After each release, you should update our [Docker repository](https://github.com/decidim/docker) so new images are build for the new release. To do it, just update `DECIDIM_VERSION` at [circle.yml](https://github.com/decidim/docker/blob/master/circle.yml).
