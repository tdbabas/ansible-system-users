# System Users (`system-users`)

Master:
[![Build Status](https://semaphoreci.com/api/v1/projects/7e62cbe6-b342-4927-953d-b9e421d59b67/600994/badge.svg)](https://semaphoreci.com/antarctica/ansible-system-users)

Develop:
[![Build Status](https://semaphoreci.com/api/v1/projects/7e62cbe6-b342-4927-953d-b9e421d59b67/593849/badge.svg)](https://semaphoreci.com/antarctica/ansible-system-users)

Creates one or more operating system users with various optional attributes.

**Part of the BAS Ansible Role Collection (BARC)**

## Overview

* Creates, or removes, one or more operating system users accounts
* Optionally, sets a number of user account attributes such as UID, comment, shell, etc.
* Optionally, adds more or more public key files to the `authorized_keys` file of a user account
* Optionally, sets the primary group of a user account to an existing operating system group
* Optionally, sets the secondary groups for a user account to one or more existing operating system groups

## Quality Assurance

This role uses manual and automated testing to ensure the features offered by this role work as advertised. 
See `tests/README.md` for more information.

## Dependencies

* None

## Requirements

* If adding public keys to a user account, these must already exist on your local machine
* If setting the primary group or secondary groups for a user account, these must already exist as operating system
groups

## Limitations

* The `authorized_keys_files` option must be specified for all system users, regardless of whether the authorized_keys 
feature is used for that user

This occurs because of the 
[sub-elements loop](http://docs.ansible.com/ansible/playbooks_loops.html#looping-over-subelements) used for managing
authorised keys. An [issue](https://github.com/ansible/ansible/issues/9827) is open for this exact problem, but is 
unlikely to be fixed until Ansible 2.0.

Where you are not managing the authorised keys for a user with this role, this variable can be safely set to an empty
list. This role will only apply changes to a users authorised keys file where the `authorized_keys_directory` is 
defined for a user.

E.g.

```yaml
---

# In this example the user 'user' does not have authorised keys managed by this role as the 'authorized_keys_directory'
# option is missing. To prevent Ansible complaining about an undefined variable used in a sub-elements loop, an empty
# list is set for 'authorized_keys_files' option.

# System Users
system_users_users:
  -
    username: user
    authorized_keys_files: []
```

*This limitation is considered to be significantly limiting, and a solution will be actively pursued. Pull requests 
to address this will be gratefully considered and given priority.*

See [BARC-62](https://jira.ceh.ac.uk/browse/BARC-62) for further details.

* Authorised keys must be expressed as individual public key files inside the same directory

It is not possible to use all files within the directory set by `authorized_keys_directory`, or to set public keys 
using in-line values. It is also not possible to use public key files contained in multiple directories due to a 
hard-coded [file lookup](http://docs.ansible.com/ansible/playbooks_lookups.html#intro-to-lookups-getting-file-contents) 
in the form: `lookup('file', [authorized_keys_directory] + '/' + [authorized_key])`.

This means you cannot use the contents of a directory to dynamically control which public keys will be added as
authorised keys. Instead you would also need to manage the `authorized_keys_files` list accordingly.

*This limitation is considered to be significantly limiting, and a solution will be actively pursued. Pull requests 
to address this will be gratefully considered and given priority.*

See [BARC-61](https://jira.ceh.ac.uk/browse/BARC-61) for further details.

* The authorized_keys file must be located at the conventional path `~/.ssh/authorized_keys`

This conventional path is currently hard-coded (as `/home/[username]/.ssh/authorized_keys`) so that role tests can
ensure the ownership and permissions of the `.ssh` directory and `.ssh/authorized_keys` file are properly set.

*This limitation is **not** considered to be significantly limiting, and a solution will not be actively pursued. Pull 
requests addressing this will be considered however.*

See [BARC-63](https://jira.ceh.ac.uk/browse/BARC-63) for further details.

* Authorised keys cannot be removed, only added

It is not possible to indicate whether a public key should be added or removed from a users authorized_keys file. It is
assumed keys will always be added.

*This limitation is considered to be significantly limiting, and a solution will be actively pursued. Pull requests 
to address this will be gratefully considered and given priority.*

See [BARC-64](https://jira.ceh.ac.uk/browse/BARC-64) for further details.

## Usage

### Typical playbook

```yaml
---

- name: setup system users
  hosts: all
  become: yes
  vars:
    system_users_users:
      -
        username: conwat
        comment: User account for Connie Watson
        shell: /bin/bash
        authorized_keys_directory: "../public_keys"
        authorized_keys_files:
          - "conwat_id_rsa.pub"
        secondary_groups:
          - adm
        state: present
  roles:
    - system-users
```

### Setting default options

It is not possible to set a 'default' user from which options are inherited [1], however it is possible to re-use 
variables yourself.

E.g.

```yaml
---

system_users_defaults:
  - shell: /bin/bash

system_users_users:
  -
    username: user1
    shell: "{{ system_users_defaults.shell }}"
    state: present
  -
    username: user2
    shell: "{{ system_users_defaults.shell }}"
    state: present
```

[1] It is not possible to chain the `default()` and `omit` filters together, 
e.g. `username: instance_username | default(default_username) | omit`.

### Tags

BARC roles use standardised tags to control which aspects of an environment are changed by roles. Where relevant, tags
will be applied at a role, or task(s) level, as indicated below.

This role uses the following tags, for all tasks:

* [**BARC_CONFIGURE**](https://antarctica.hackpad.com/BARC-Standardised-Tags-AviQxxiBa3y#:h=BARC_CONFIGURE)
* [**BARC_CONFIGURE_SYSTEM**](https://antarctica.hackpad.com/BARC-Standardised-Tags-AviQxxiBa3y#:h=BARC_CONFIGURE_SYSTEM)
* [**BARC_CONFIGURE_USERS**](https://antarctica.hackpad.com/BARC-Standardised-Tags-AviQxxiBa3y#:h=BARC_CONFIGURE_USERS)

### Variables

#### `system_users_users`

A list of operating system user accounts, and their properties, to be managed by this role.

Structured as a list of items, with each item having the following properties:

* *username*
  * **MUST** be specified
  * Values **MUST** be valid user usernames, as determined by the operating system
  * Example: `conwat`
* *uid*
  * **MAY** be specified
  * Specifies whether the user should have a fixed UID, or if should be allocated by the operating system
  * Values **MUST** be valid user UIDs, as determined by the operating system
  * Values **SHOULD** respect operating system conventions for system/non-system users and reserved or special ranges
  * Where not specified, the operating system will assign a suitable UID (i.e. the next in the relevant range)
  * Specifying this property is non-default behaviour
  * Example: `1001`
* *comment*
  * **SHOULD** be specified
  * Values **MUST** be valid user comments, as determined by the operating system
  * Values **SHOULD** be suitably descriptive to aid other users, without being needlessly verbose
  * Where not specified, no comment will be added
  * Example: `User account for Connie Watson`
* *shell*
  * **MAY** be specified
  * Specifies the shell interpreter for a user
  * Values **MUST** be valid shell interpreters (i.e. installed), as determined by the operating system
  * Where not specified, the default interpreter will be used
  * Example: `/bin/bash`
* *authorized_keys_directory*
  * **MAY** be specified if it desired for authorised keys to a added to a user account, otherwise this **MUST NOT** be 
  specified
  * Specifies in which directory the public key files indicated by the *authorized_keys_files* property are located
  * The presence of this property is used to determine whether authorised keys should be managed for the user, i.e. if
  this property is defined, the values in the *authorized_keys_files* property will be used
  * Values **MUST** represent a valid path to a directory, on the Ansible control machine (i.e. localhost), 
  values **MUST** not use a trailing directory separator (i.e. `/`)
  * Example `../public_keys` - assuming this role is within a `roles` directory, this would point to a `public_keys` 
  directory that sits alongside the roles directory
* *authorized_keys_files*
  * **MUST** be specified as a property (see limitations above), however items **MAY** be specified 
  * Specifies the individual public key files that should be added the user's `authorized_keys` file
  * Structured as a list of items, with each item representing the file name of a public key file
  * Item values **MUST** represent valid file names for valid SSH public key files, as determined by SSH, located within 
  the directory set by the *authorized_keys_directory* property
  * Where no public keys should be added for a user, this property can be set to an empty list (i.e. `[]`)
  * Example: `- "conwat_id_rsa.pub"` - of a single item within this property
* *primary_group*
  * **MAY** be specified
  * Specifies whether a pre-existing, named, group should be used for the user's primary group, or if a group named
  after the user's username should be created and used
  * Values **MUST** be the name for a valid group, that already exists, as determined by the operating system
  * Where not specified, the operating system will create a group based on the user's username and use this as the
  primary group
  * Specifying this property is non-default behaviour, it is much more likely you will want to use secondary groups 
  instead
  * Example: `foo`
* *secondary_groups*
  * **MAY** be specified
  * Specifies supplementary (secondary), pre-existing, named, groups the user should be member of
  * Structured as a list of items, with each item representing a secondary group
  * Item values **MUST** represent the name of valid group, that already exists, as determined by the operating system
  * Where no secondary groups should be added for a user, this property can be omitted
  * Example: `- adm` - of a single item within this property
* *state*
  * **SHOULD** be specified
  * Specifies whether the user should be present as user, or absent
  * Values **MUST** use one of these options, as determined by Ansible:
    * `present`
    * `absent`
  * Where not specified it will be assumed the user should be present
  * Example: `present`

Default value: `[]` - an empty list

## Developing

### Issue tracking

Issues, bugs, improvements, questions, suggestions and other tasks related to this package are managed through the 
[BAS Ansible Role Collection](https://jira.ceh.ac.uk/projects/BARC) (BARC) project on Jira.

This service is currently only available to BAS or NERC staff, although external collaborators can be added on request.
See our contributing policy for more information.

### Source code

All changes should be committed, via pull request, to the canonical repository, which for this project is:

`ssh://git@stash.ceh.ac.uk:7999/barc/system-users.git`

A mirror of this repository is maintained on GitHub. Changes are automatically pushed from the canonical repository to
this mirror, in a one-way process.

`git@github.com:antarctica/ansible-system-users.git`

Note: The canonical repository is only accessible within the NERC firewall. External collaborators, please make pull 
requests against the mirrored GitHub repository and these will be merged as appropriate.

### Contributing policy

This project welcomes contributions, see `CONTRIBUTING.md` for our general policy.

The [Git flow](https://github.com/antarctica/ansible-apache/blob/develop/sian.com/git/tutorials/comparing-workflows/gitflow-workflow) 
workflow is used to manage the development of this project:

* Discrete changes should be made within feature branches, created from and merged back into develop 
(where small changes may be made directly)
* When ready to release a set of features/changes, create a release branch from develop, update documentation as 
required and merge into master with a tagged, semantic version (e.g. v1.2.3)
* After each release, the master branch should be merged with develop to restart the process
* High impact bugs can be addressed in hotfix branches, created from and merged into master (then develop) directly

## Licence

Copyright 2015 NERC BAS.

Unless stated otherwise, all documentation is licensed under the Open Government License - version 3. All code is
licensed under the MIT license.

Copies of these licenses are included within this role.
