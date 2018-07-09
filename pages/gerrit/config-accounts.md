---
title: " Gerrit Code Review - Accounts"
sidebar: gerritdoc_sidebar
permalink: config-accounts.html
---
## Overview

Starting from 2.15 Gerrit accounts are fully stored in
[NoteDb](dev-note-db.html).

The account data consists of a sequence number (account ID), account
properties (full name, preferred email, registration date, status,
inactive flag), preferences (general, diff and edit preferences),
project watches, SSH keys, external IDs, starred changes and reviewed
flags.

Most account data is stored in a special [All-Users](#all-users)
repository, which has one branch per user. Within the user branch there
are Git config files for the [account properties](#account-properties),
the [account preferences](#preferences) and the [project
watches](#project-watches). In addition there is an `authorized_keys`
file for the [SSH keys](#ssh-keys) that follows the standard OpenSSH
file format.

The account data in the user branch is versioned and the Git history of
this branch serves as an audit log.

The [external IDs](#external-ids) are stored as Git Notes inside the
`All-Users` repository in the `refs/meta/external-ids` notes branch.
Storing all external IDs in a notes branch ensures that each external ID
is only used once.

The [starred changes](#starred-changes) are represented as independent
refs in the `All-Users` repository. They are not stored in the user
branch, since this data doesn’t need versioning.

The [reviewed flags](#reviewed-flags) are not stored in Git, but are
persisted in a database table. This is because there is a high volume of
reviewed flags and storing them in Git would be inefficient.

Since accessing the account data in Git is not fast enough for account
queries, e.g. when suggesting reviewers, Gerrit has a [secondary index
for accounts](#account-index).

## `All-Users` repository

The `All-Users` repository is a special repository that only contains
user-specific information. It contains one branch per user. The user
branch is formatted as `refs/users/CD/ABCD`, where `CD/ABCD` is the
[sharded account ID](access-control.html#sharded-user-id), e.g. the user
branch for account `1000856` is `refs/users/56/1000856`. The account IDs
in the user refs are sharded so that there is a good distribution of the
Git data in the storage system.

A user branch must exist for each account, as it represents the account.
The files in the user branch are all optional. This means having a user
branch with a tree that is completely empty is also a valid account
definition.

Updates to the user branch are done through the [Gerrit REST
API](rest-api-accounts.html), but users can also manually fetch their
user branch and push changes back to Gerrit. On push the user data is
evaluated and invalid user data is rejected.

To hide the implementation detail of the sharded account ID in the ref
name Gerrit offers a magic `refs/users/self` ref that is automatically
resolved to the user branch of the calling user. The user can then use
this ref to fetch from and push to the own user branch. E.g. if user
`1000856` pushes to `refs/users/self`, the branch
`refs/users/56/1000856` is updated. In Gerrit `self` is an established
term to refer to the calling user (e.g. in change queries). This is why
the magic ref for the own user branch is called `refs/users/self`.

A user branch should only be readable and writeable by the user to whom
the account belongs. To assign permissions on the user branches the
normal branch permission system is used. In the permission system the
user branches are specified as `refs/users/${shardeduserid}`. The
`${shardeduserid}` variable is resolved to the sharded account ID. This
variable is used to assign default access rights on all user branches
that apply only to the owning user. The following permissions are set by
default when a Gerrit site is newly installed or upgraded to a version
which supports user branches:

**All-Users project.config.**

    [access "refs/users/${shardeduserid}"]
      exclusiveGroupPermissions = read push submit
      read = group Registered Users
      push = group Registered Users
      label-Code-Review = -2..+2 group Registered Users
      submit = group Registered Users

The user branch contains several files with account data which are
described [below](#account-data-in-user-branch).

In addition to the user branches the `All-Users` repository also
contains a branch for the [external IDs](#external-ids) and special refs
for the [starred changes](#starred-changes).

Also the next available value of the [account
sequence](#account-sequence) is stored in the `All-Users` repository.

## Account Index

There are several situations in which Gerrit needs to query accounts,
e.g.:

  - For sending email notifications to project watchers.

  - For reviewer suggestions.

Accessing the account data in Git is not fast enough for account
queries, since it requires accessing all user branches and parsing all
files in each of them. To overcome this Gerrit has a secondary index for
accounts. The account index is either based on [Lucene or
Elasticsearch](config-gerrit.html#index.type).

Via the [Query Account](rest-api-accounts.html#query-account) REST
endpoint [generic account queries](user-search-accounts.html) are
supported.

Accounts are automatically reindexed on any update. The [Index
Account](rest-api-accounts.html#index-account) REST endpoint allows to
reindex an account manually. In addition the [reindex](pgm-reindex.html)
program can be used to reindex all accounts offline.

## Account Data in User Branch

A user branch contains several Git config files with the account data:

  - `account.config`:
    
    Stores the [account properties](#account-properties).

  - `preferences.config`:
    
    Stores the [user preferences](#preferences) of the account.

  - `watch.config`:
    
    Stores the [project watches](#project-watches) of the account.

In addition it contains an
[authorized\_keys](https://en.wikibooks.org/wiki/OpenSSH/Client_Configuration_Files#.7E.2F.ssh.2Fauthorized_keys)
file with the [SSH keys](#ssh-keys) of the account.

### Account Properties

The account properties are stored in the user branch in the
`account.config` file:

    [account]
      fullName = John Doe
      preferredEmail = john.doe@example.com
      status = OOO
      active = false

For active accounts the `active` parameter can be omitted.

The registration date is not contained in the `account.config` file but
is derived from the timestamp of the first commit on the user branch.

When users update their account properties by pushing to the user
branch, it is verified that the preferred email exists in the external
IDs.

Users are not allowed to flip the active value themselves; only
administrators and users with the [Modify
Account](access-control.html#capability_modifyAccount) global capability
are allowed to change it.

Since all data in the `account.config` file is optional the
`account.config` file may be absent from some user branches.

### Preferences

The account properties are stored in the user branch in the
`preferences.config` file. There are separate sections for
[general](intro-user.html#preferences),
[diff](user-review-ui.html#diff-preferences) and edit preferences:

    [general]
      showSiteHeader = false
    [diff]
      hideTopMenu = true
    [edit]
      lineLength = 80

The parameter names match the names that are used in the preferences
REST API:

  - [General Preferences](rest-api-accounts.html#preferences-info)

  - [Diff Preferences](rest-api-accounts.html#diff-preferences-info)

  - [Edit Preferences](rest-api-accounts.html#edit-preferences-info)

If the value for a preference is the same as the default value for this
preference, it can be omitted in the `preference.config` file.

Defaults for general and diff preferences that apply for all accounts
can be configured in the `refs/users/default` branch in the `All-Users`
repository.

### Project Watches

Users can configure watches on projects to receive email notifications
for changes of that project.

A watch configuration consists of the project name and an optional
filter query. If a filter query is specified, email notifications will
be sent only for changes of that project that match this query.

In addition, each watch configuration can contain a list of notification
types that determine for which events email notifications should be
sent. E.g. a user can configure that email notifications should only be
sent if a new patch set is uploaded and when the change gets submitted,
but not on other events.

Project watches are stored in a `watch.config` file in the user branch:

    [project "foo"]
      notify = * [ALL_COMMENTS]
      notify = branch:master [ALL_COMMENTS, NEW_PATCHSETS]
      notify = branch:master owner:self [SUBMITTED_CHANGES]

The `watch.config` file has one project section for all project watches
of a project. The project name is used as subsection name and the
filters with the notification types, that decide for which events email
notifications should be sent, are represented as `notify` values in the
subsection. A `notify` value is formatted as "\<filter\>
\[\<comma-separated-list-of-notification-types\>\]". The supported
notification types are described in the [Email Notifications
documentation](user-notify.html#notify.name.type).

For a change event, a notification will be sent if any `notify` value of
the corresponding project has both a filter that matches the change and
a notification type that matches the event.

In order to send email notifications on change events, Gerrit needs to
find all accounts that watch the corresponding project. To make this
lookup fast the secondary account index is used. The account index
contains a repeated field that stores the projects that are being
watched by an account. After the accounts that watch the project have
been retrieved from the index, the complete watch configuration is
available from the account cache and Gerrit can check if any watch
matches the change and the event.

### SSH Keys

SSH keys are stored in the user branch in an `authorized_keys` file,
which is the [standard OpenSSH file
format](https://en.wikibooks.org/wiki/OpenSSH/Client_Configuration_Files#.7E.2F.ssh.2Fauthorized_keys)
for storing SSH
    keys:

    ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAAgQCgug5VyMXQGnem2H1KVC4/HcRcD4zzBqSuJBRWVonSSoz3RoAZ7bWXCVVGwchtXwUURD689wFYdiPecOrWOUgeeyRq754YWRhU+W28vf8IZixgjCmiBhaL2gt3wff6pP+NXJpTSA4aeWE5DfNK5tZlxlSxqkKOS8JRSUeNQov5Tw== john.doe@example.com
    # DELETED
    # INVALID ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAAgQDm5yP7FmEoqzQRDyskX+9+N0q9GrvZeh5RG52EUpE4ms/Ujm3ewV1LoGzc/lYKJAIbdcZQNJ9+06EfWZaIRA3oOwAPe1eCnX+aLr8E6Tw2gDMQOGc5e9HfyXpC2pDvzauoZNYqLALOG3y/1xjo7IH8GYRS2B7zO/Mf9DdCcCKSfw== john.doe@example.com
    ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAAAgQCaS7RHEcZ/zjl9hkWkqnm29RNr2OQ/TZ5jk2qBVMH3BgzPsTsEs+7ag9tfD8OCj+vOcwm626mQBZoR2e3niHa/9gnHBHFtOrGfzKbpRjTWtiOZbB9HF+rqMVD+Dawo/oicX/dDg7VAgOFSPothe6RMhbgWf84UcK5aQd5eP5y+tQ== john.doe@example.com

When the SSH API is used, Gerrit needs an efficient way to lookup SSH
keys by username. Since the username can be easily resolved to an
account ID (via the account cache), accessing the SSH keys in the user
branch is fast.

To identify SSH keys in the REST API Gerrit uses [sequence numbers per
account](rest-api-accounts.html#ssh-key-id). This is why the order of
the keys in the `authorized_keys` file is used to determines the
sequence numbers of the keys (the sequence numbers start at 1).

To keep the sequence numbers intact when a key is deleted, a *\#
DELETED* line is inserted at the position where the key was deleted.

Invalid keys are marked with the prefix *\# INVALID*.

## External IDs

External IDs are used to link external identities, such as an LDAP
account or an OAUTH identity, to an account in Gerrit.

External IDs are stored as Git Notes in the `All-Users` repository. The
name of the notes branch is `refs/meta/external-ids`.

As note key the SHA1 of the external ID key is used. This ensures that
an external ID is used only once (e.g. an external ID can never be
assigned to multiple accounts at a point in time).

The note content is a Git config file:

    [externalId "username:jdoe"]
      accountId = 1003407
      email = jdoe@example.com
      password = bcrypt:4:LCbmSBDivK/hhGVQMfkDpA==:XcWn0pKYSVU/UJgOvhidkEtmqCp6oKB7

The config file has one `externalId` section. The external ID key which
consists of scheme and ID in the format *\<scheme\>:\<id\>* is used as
subsection name.

The `accountId` field is mandatory, the `email` and `password` fields
are optional.

The external IDs are maintained by Gerrit, this means users are not
allowed to manually edit their external IDs. Only users with the [Access
Database](access-control.html#capability_accessDatabase) global
capability can push updates to the `refs/meta/external-ids` branch.
However Gerrit rejects pushes if:

  - any external ID config file cannot be parsed

  - if a note key does not match the SHA of the external ID key in the
    note content

  - external IDs for non-existing accounts are contained

  - invalid emails are contained

  - any email is not unique (the same email is assigned to multiple
    accounts)

  - hashed passwords of external IDs with scheme `username` cannot be
    decoded

## Starred Changes

[Starred changes](dev-stars.html) allow users to mark changes as
favorites and receive email notifications for them.

Each starred change is a tuple of an account ID, a change ID and a
label.

To keep track of a change that is starred by an account, Gerrit creates
a `refs/starred-changes/YY/XXXX/ZZZZZZZ` ref in the `All-Users`
repository, where `YY/XXXX` is the sharded numeric change ID and
`ZZZZZZZ` is the account ID.

A starred-changes ref points to a blob that contains the list of labels
that the account set on the change. The label list is stored as UTF-8
text with one label per line.

Since JGit has explicit optimizations for looking up refs by prefix when
the prefix ends with */*, this ref format is optimized to find starred
changes by change ID. Finding starred changes by change ID is e.g.
needed when a change is updated so that all users that have the [default
star](dev-stars.html#default-star) on the change can be notified by
email.

Gerrit also needs an efficient way to find all changes that were starred
by an account, e.g. to provide results for the
[is:starred](user-search.html#is-starred) query operator. With the ref
format as described above the lookup of starred changes by account ID is
expensive, as this requires a scan of the full `refs/starred-changes/*`
namespace. To overcome this the users that have starred a change are
stored in the change index together with the star labels.

## Reviewed Flags

When reviewing a patch set in the Gerrit UI, the reviewer can mark files
in the patch set as reviewed. These markers are called ‘Reviewed Flags’
and are private to the user. A reviewed flag is a tuple of patch set ID,
file and account ID.

Each user can have many thousands of reviewed flags and over time the
number can grow without bounds.

The high amount of reviewed flags makes a storage in Git unsuitable
because each update requires opening the repository and committing a
change, which is a high overhead for flipping a bit. Therefore the
reviewed flags are stored in a database table. By default they are
stored in a local H2 database, but there is an extension point that
allows to plug in alternate implementations for storing the reviewed
flags. To replace the storage for reviewed flags a plugin needs to
implement the
[AccountPatchReviewStore](dev-plugins.html#account-patch-review-store)
interface. E.g. to support a multi-master setup where reviewed flags
should be replicated between the master nodes one could implement a
store for the reviewed flags that is based on MySQL with replication.

## Account Sequence

The next available account sequence number is stored as UTF-8 text in a
blob pointed to by the `refs/sequences/accounts` ref in the `All-Users`
repository.

Multiple processes share the same sequence by incrementing the counter
using normal git ref updates. To amortize the cost of these ref updates,
processes increment the counter by a larger number and hand out numbers
from that range in memory until they run out. The size of the account ID
batch that each process retrieves at once is controlled by the
[notedb.accounts.sequenceBatchSize](config-gerrit.html#notedb.accounts.sequenceBatchSize)
parameter in the `gerrit.config` file.

## GERRIT

Part of [Gerrit Code Review](index.html)

## SEARCHBOX
