First, verify that you meet all the [prerequisites](https://github.com/coreos/fedora-coreos-streams/blob/master/release-prereqs.md)

Name this issue `next: new release on YYYY-MM-DD` with today's date. Once the pipeline spits out the new version ID, you can append it to the title e.g. `(31.20191117.2.0)`.

# Pre-release

## Promote next-devel changes to next

From the checkout for `fedora-coreos-config` (replace `upstream` below with
whichever remote name tracks `coreos/`):

- [ ] `git fetch upstream`
- [ ] `git checkout next`
- [ ] `git reset --hard upstream/next`
- [ ] `/path/to/fedora-coreos-releng-automation/scripts/promote-config.sh next-devel`
- [ ] Sanity check promotion with `git show`
- [ ] Open PR against the `next` branch on https://github.com/coreos/fedora-coreos-config
- [ ] Post a link to the PR as a comment to this issue
- [ ] Ideally have at least one other person check it and approve
- [ ] Once CI has passed, merge it

## Build

- [ ] Start a [pipeline build](https://jenkins-fedora-coreos.apps.ocp.ci.centos.org/job/fedora-coreos/job/fedora-coreos-fedora-coreos-pipeline/build?delay=0sec) (select `next`, leave all other defaults)
- [ ] Post a link to the job as a comment to this issue
- [ ] Wait for the job to finish

## Sanity-check the build

Using the [the build browser](https://builds.coreos.fedoraproject.org/browser) for the `next` stream:

- [ ] Verify that the parent commit and version match the previous `next` release (in the future, we'll want to integrate this check in the release job)
- [ ] Check [kola AWS run](https://jenkins-fedora-coreos.apps.ocp.ci.centos.org/job/kola-aws/) to make sure it didn't fail
- [ ] Check [kola GCP run](https://jenkins-fedora-coreos.apps.ocp.ci.centos.org/job/kola-gcp/) to make sure it didn't fail
- [ ] Check [kola OpenStack run](https://jenkins-fedora-coreos.apps.ocp.ci.centos.org/job/kola-openstack/) to make sure it didn't fail

# ⚠️ Release ⚠️

IMPORTANT: this is the point of no return here. Once the OSTree commit is
imported into the unified repo, any machine that manually runs `rpm-ostree
upgrade` will have the new update.

## Run the release job

- [ ] Run the [release job](https://jenkins-fedora-coreos.apps.ocp.ci.centos.org/job/fedora-coreos/job/fedora-coreos-fedora-coreos-pipeline-release/build?delay=0sec), filling in for parameters `next` and the new version ID
- [ ] Post a link to the job as a comment to this issue
- [ ] Wait for job to finish
- [ ] Verify that the OSTree commit and its signature are present and valid by booting a VM at the previous release (e.g. `cosa run --qemu-image /path/to/previous.qcow2`) and verifying that `rpm-ostree upgrade` works and `rpm-ostree status` shows a valid signature.

At this point, Cincinnati will see the new release on its next refresh and create a corresponding node in the graph without edges pointing to it yet.

## Refresh metadata (stream and updates)

From a checkout of this repo:

- [ ] Update stream metadata, by running:


```
fedora-coreos-stream-generator -releases=https://fcos-builds.s3.amazonaws.com/prod/streams/next/releases.json  -output-file=streams/next.json -pretty-print
```

- Update the updates metadata, editing `updates/next.json`:
  - [ ] Find the last-known-good release (whose `rollout` has a `start_percentage` of `1.0`) and set its `version` to the most recent completed rollout
  - [ ] Delete releases with completed rollouts
  - Add a new rollout:
    - [ ] Set `version` field to the new version
    - [ ] Set `start_epoch` field to a future timestamp for the rollout start (e.g. `date -d '20yy/mm/dd 14:30UTC' +%s`)
    - [ ] Set `start_percentage` field to `0.0`
    - [ ] Set `duration_minutes` field to a reasonable rollout window (e.g. `2880` for 48h)
  - [ ] Update the `last-modified` field to current time (e.g. `date -u +%Y-%m-%dT%H:%M:%SZ`)

A reviewer can validate the `start_epoch` time by running `date -u -d @<EPOCH>`. An example of encoding and decoding in one step: `date -d '2019/09/10 14:30UTC' +%s | xargs -I{} date -u -d @{}`. 

- [ ] Commit the changes and open a PR against the repo.
- [ ] Post a link to the PR as a comment to this issue
- [ ] Wait for the PR to be approved.
- [ ] Once approved, merge it and verify that the [`sync-stream-metadata` job](https://jenkins-fedora-coreos.apps.ocp.ci.centos.org/job/sync-stream-metadata/) syncs the contents to S3
- [ ] Verify the new version shows up on [the download page](https://getfedora.org/en/coreos/download?stream=next)
- [ ] Verify the incoming edges are showing up in the [update graph](https://builds.coreos.fedoraproject.org/graph?stream=next)

<details>
  <summary>Update graph manual check</summary>

```
curl -H 'Accept: application/json' 'https://updates.coreos.fedoraproject.org/v1/graph?basearch=x86_64&stream=next&rollout_wariness=0'
```

</details>

NOTE: In the future, most of these steps will be automated.

## Housekeeping

- [ ] If one doesn't already exist, [open an issue](https://github.com/coreos/fedora-coreos-streams/issues/new?labels=kind/release,jira&template=next.md) in this repo for the next release in this stream. Use the approximate date of the release in the title.
- [ ] Issues opened via the previous link will automatically create a linked Jira card. Assign the GitHub issue and Jira card to the next person in the [rotation](https://hackmd.io/WCA8XqAoRvafnja01JG_YA).
- [ ] Check the overrides lockfiles in the configs repo for the `next-devel` stream to see if any overrides are obsolete. They are obsolete if the RPMs (or newer ones) have hit the stable Fedora repos. You can usually see this by following the Bodhi link in the lockfile and checking whether the update was pushed to stable or was obsoleted by an update which was pushed to stable.
  - If a PR was created post a link to the PR as a comment to this issue.
