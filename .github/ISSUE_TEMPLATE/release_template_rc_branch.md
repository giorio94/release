---
name: Release a new RC version of Cilium from a stable branch
about: Create a checklist for an upcoming release
title: 'vX.Y.Z-rc.W release'
labels: kind/release
assignees: ''

---

## Setup preparation

- [ ] Depending on your OS, make sure Docker is running
- [ ] Export a [`GITHUB_TOKEN`](https://github.com/settings/tokens/new?description=Cilium%20Release%20Script&scopes=write:org,public_repo) that has access to the repository
- [ ] Make sure a setup (GPG, SSH, S/MIME) is in place for [signing tags] with Git
- [ ] Make sure the `GOPATH` environment variable is set and pointing to the relevant path
- [ ] Make sure the [Cilium helm charts][Cilium charts] repository is installed locally:
  - [ ] Run `git clone https://github.com/cilium/charts.git "$GOPATH/src/github.com/cilium/charts"`

## Pre-release


- [ ] Announce in Cilium slack channel #launchpad: `Starting vX.Y.Z-rc.W release process :ship:`
- [ ] Create a thread for that message and ping current top-hat to not merge any
  PRs until the release process is complete.
- [ ] Change directory to the local copy of Cilium repository.
- [ ] Check that there are no [release blockers] for the targeted release version
- [ ] Ensure that outstanding [backport PRs] are merged
- [ ] If stable branch is not created yet. Run:
  - `git fetch upstream && git checkout -b upstream/vX.Y upstream/main`
  - [ ] Push that branch to GitHub:
    - `git push upstream vX.Y`
  - [ ] On the main branch, create a PR with a change in the `VERSION` file to
        start the next development cycle as well as creating the necessary GH
        workflows (renovate configuration, MLH configuration, etc.
        see [24143732b616](https://github.com/cilium/cilium/commit/24143732b616bb6cd308564b0be33f13fc5613e6)
        for reference):
    - [ ] Adjust `maintainers-little-helper.yaml` accordingly the new stable
          branch.
    - [ ] Check for any other .github workflow references to the current stable
          branch `X.Y-1`, and update those to include the new stable `X.Y`
          version as well.
        - `git grep "X.Y-1" .github/`
    - [ ] Ensure that the `CustomResourceDefinitionSchemaVersion` uses a new
          minor schema version compared to the new `X.Y` release.
    - `echo "X.Y+1.0-dev" > VERSION`
    - `make -C install/kubernetes`
    - `git add .github/ Documentation/contributing/testing/ci.rst`
    - `git commit -sam "Prepare for vX.Y+1 development cycle"`
  - [ ] Merge the main PR so that the stable branch protection can be properly
        set up with the right status checks requirements.
  - [ ] Sync the `vX.Y` branch up to the commit before preparing for the `X.Y+1` development cycle.
    - `git fetch upstream && git checkout vX.Y && git merge --ff-only upstream/main~1 && git log -5`
    - `git push upstream vX.Y`
  - [ ] Protect the new stable branch with GitHub Settings [here](https://github.com/cilium/cilium/settings/branches)
      - Use the settings of the previous stable branch and main as sane defaults
  - [ ] On the `vX.Y` branch, prepare for stable release development:
    - [ ] Update the VERSION file with the last prerelease for this stable version
      - `echo "X.Y.Z-pre.N" > VERSION`
    - [ ] Remove any GitHub workflows from the stable branch that are only
          relevant for the main branch.
      - Remove workflows that are exclusively triggered by cron job and
        workflows triggered by `issue_comment` triggers, as they do not run on
        stable branches. These can be identified with commands like this:
        - `git grep -B 5 cron .github/ | grep name | sed 's/-name.*//g'`
        - `git grep issue_comment .github/`
      - Replace references to `main` branch with `X.Y` in the workflows.
        - `sed -i 's/- \(ft\/\)\?main/- \1vX.Y/g' .github/workflows/*`
        - `sed -i 's/@main/@vX.Y/g' .github/workflows/*`
        - `sed -i 's/\/main\//\/vX.Y\//g' .github/workflows/*`
      - Ensure that the `.github/workflows/call-backport-label-updater.yaml` workflow
        includes the `vX.Y` version in the `branch` matrix list.
        (as an example, look at the
        [workflow](https://github.com/cilium/cilium/blob/9c3694f3ae9f8472d4d6a2a32cf506f77923be53/.github/workflows/call-backport-label-updater.yaml#L16)
        in the v1.16 stable branch: `v1.16` is included in the list).
        If `vX.Y` is missing, update the workflow removing the oldest
        stable version and adding `vX.Y`.
      - Ensure that the `CustomResourceDefinitionSchemaVersion` uses a new
        minor schema version compared to the previous stable release.
      - Update `install/kubernetes/Makefile*`, following the changes made
        during the previous stable branch preparation commit on the previous
        stable branch.
      - Remove `stable.txt` file
      - You may want to initially commit the state up until now before the next
        step, so that it's easier to compare the diff vs. the previous stable
        release.
      - Copy-paste the `.github` directory from the previous stable branch and
        manually check the diff between the files from the current stable branch
        and modify the workflows to match the target stable branch. See
        [8bbae9cb43](https://github.com/cilium/cilium/commit/8bbae9cb4323bf3dd94936e355b0c2aad96d0df8)
        for reference.
      - Ignore all stable branch changes under the `.github/actions` directory.
        `git checkout .github/actions`
      - Remove the `labels-unset` field from the MLH configuration and add
        the `auto-label` field. See [5b4934284d](https://github.com/cilium/cilium/commit/5b4934284dd525399aacec17c137811df9cf0f8b)
        for reference.
      - Rewrite the CODEOWNERS file. Keep the team descriptions from main
        and the previous stable branch. See [97daf56221](https://github.com/cilium/cilium/commit/97daf5622197d0cdda003a3f693e6e5a61038884)
      - Update CODEOWNERS documentation file by running `make -C Documentation update-codeowners`
      - Replace references to `bpf-next-*` lvh images in workflows with the newest LTS kernel from [quay.io](https://quay.io/repository/lvh-images/kind?tab=tags&tag=latest).
        `grep -R bpf-next- .github/workflows/`
    - [ ] Review the diff for this commit compared to the preparation commit
          for the previous stable branch.
    - [ ] Push a PR with those changes:
      - `git commit -sam "Prepare vX.Y stable branch"`
      - `gh pr create -B vX.Y`
- [ ] Push a PR including the changes necessary for the new release:
  - [ ] Run `../release/internal/start-release.sh vX.Y.Z-rc.W`
        Note that this script produces some files at the root of the Cilium
        repository, and that these files are required at a later step for
        tagging the release.
  - [ ] Check the modified schema file(s) in `Documentation` as it will be
        necessary to fix them manually. Add a new line for this RC and remove
        unsupported versions.
  - [ ] Fix any duplicate `AUTHORS` entries and verify if it is possible to
        get the real names instead of GitHub usernames.
  - [ ] Add the generated `CHANGELOG.md` file and commit all remaining changes
        with the title `Prepare for release vX.Y.Z-rc.W`
  - [ ] Submit PR (`../release/internal/submit-release.sh`)
- [ ] Merge PR
- [ ] Ping current top-hat that PRs can be merged again.
- [ ] Create and push *both* tags to GitHub (`vX.Y.Z-rc.W`, `X.Y.Z-rc.W`)
  - Pull latest branch locally.
  - Check out the release commit and run `../release/internal/tag-release.sh`
    against that commit.
- [ ] Ask a maintainer to approve the build in the following link (keep the URL
      of the GitHub run to be used later):
      [Cilium Image Release builds](https://github.com/cilium/cilium/actions?query=workflow:%22Image+Release+Build%22)
  - [ ] Check if all docker images are available before announcing the release:
        `make -C install/kubernetes/ RELEASE=yes CILIUM_BRANCH=vX.Y check-docker-images`
- [ ] Get the image digests from the build process and make a commit and PR with
      these digests.
  - [ ] Run `../release/internal/post-release.sh URL` to fetch the image
        digests and submit a PR to update these, use the `URL` of the GitHub
        run here
  - [ ] Get someone to review the PR. Do not trigger the full CI suite, but
        wait for the automatic checks to complete. Merge the PR.
- [ ] Update helm charts
  - [ ] Create helm charts artifacts in [Cilium charts] repository using
        [cilium helm release tool] for the `vX.Y.Z-rc.W` release and push these
        changes into the helm repository.
  - [ ] Check the output of the [chart workflow] and see if the test was
        successful.
- [ ] Check [read the docs] configuration:
    - [ ] Set a new build as active and hidden in [active versions].
    - [ ] Deactivate previous RCs.
    - [ ] Update algolia configuration search in [docsearch-scraper-webhook].
      - Update the versions in `docsearch.config.json`, commit them and push a trigger the workflow [here](https://github.com/cilium/docsearch-scraper-webhook/actions/workflows/update-algolia-index.yaml)
- [ ] Check draft release from [releases] page
  - [ ] Update the text at the top with 2-3 highlights of the release
  - [ ] Publish the release
- [ ] Announce the release in #general on Slack.
  Text template for the first RC:
```
*Announcement* :tada: :tada:

:cilium-new: *Cilium release candidate vX.Y.Z-rc.W has been released:*
https://github.com/cilium/cilium/releases/tag/vX.Y.Z-rc.W

This is the first monthly snapshot for the vX.Y development cycle. There are [vX.Y.Z-rc.W OSS docs](https://docs.cilium.io/en/vX.Y.Z-rc.W) available if you want to pull this version & try it out.
```
Text template for the next RCs:
```
*Announcement* :tada: :tada:

:cilium-new: *Cilium release candidate vX.Y.Z-rc.W has been released:*
https://github.com/cilium/cilium/releases/tag/vX.Y.Z-rc.W

Thank you for the testing and contributing to the previous pre-releases. There are [vX.Y.Z-rc.W OSS docs](https://docs.cilium.io/en/vX.Y.Z-rc.W) available if you want to pull this version & try it out.
```
- [ ] Bump the development snapshot version in `README.rst` on the main branch
      to point to this release
- [ ] Prepare post-release changes to main branch using `../release/internal/bump-readme.sh`.
- [ ] Update the upgrade guide and [roadmap](https://github.com/cilium/cilium/blob/main/Documentation/community/roadmap.rst) for any features that changed status.

[signing tags]: https://docs.github.com/en/authentication/managing-commit-signature-verification/signing-tags
[release blockers]: https://github.com/cilium/cilium/labels/release-blocker%2FX.Y
[backport PRs]: https://github.com/cilium/cilium/pulls?q=is%3Aopen+is%3Apr+label%3Abackport%2FX.Y
[Cilium charts]: https://github.com/cilium/charts
[releases]: https://github.com/cilium/cilium/releases
[cilium helm release tool]: https://github.com/cilium/charts/blob/master/RELEASE.md
[cilium-runtime images]: https://quay.io/repository/cilium/cilium-runtime
[read the docs]: https://readthedocs.org/projects/cilium/
[active versions]: https://readthedocs.org/projects/cilium/versions/?version_filter=vX.Y.Z-rc.W
[docsearch-scraper-webhook]: https://github.com/cilium/docsearch-scraper-webhook
[chart workflow]: https://github.com/cilium/charts/actions/workflows/validate-cilium-chart.yaml
