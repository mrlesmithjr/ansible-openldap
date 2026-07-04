commit b7f893167654dfde65d75ec117270b695c2c4470
Author: Larry Smith Jr. <mrlesmithjr@gmail.com>
Date:   Sat Jul 4 12:20:52 2026 -0400

    ci: modernize GitHub Actions workflow and molecule scenario matrix (#18)
    
    * ci: modernize GitHub Actions workflow
    
    Bump actions/checkout v1->v4 and actions/setup-python v1->v5, drop the
    EOL Python 3.5/3.6/3.7 matrix in favor of 3.12, and remove the broken
    `sudo apt install docker` step (ubuntu-latest runners already ship
    Docker). Also drop the unused `path: ansible-openldap` checkout option,
    which put the checked-out repo in a subdirectory that none of the
    later steps (pip install, molecule) actually ran from.
    
    refs #17
    
    * fix: resolve role FQRN for current ansible-galaxy/molecule
    
    meta/main.yml had no galaxy_info.namespace, so ansible-compat fell back
    to the space-containing author string, producing an invalid computed
    FQRN ("openldap" instead of "mrlesmithjr.openldap"). Modern
    ansible-compat (used by current molecule) hard-fails on this during
    the dependency/create step, so no scenario could even start. Add the
    namespace/role_name that already match how the role is published on
    Galaxy, and update molecule/shared/converge.yml's include_role to
    reference the resulting mrlesmithjr.openldap FQRN instead of the
    literal directory name, which never matched under the symlink molecule
    creates for dependency resolution.
    
    refs #17
    
    * test: refresh molecule scenario matrix to current-gen platforms
    
    Bump debian12 and ubuntu2204 scenarios to jrei/systemd images pinned
    to their actual version tags (debian10->12, ubuntu18.04->22.04), pin
    fedora to a concrete release (42) instead of floating on :latest, and
    add a rockylinux9 scenario (using geerlingguy/docker-rockylinux9-ansible,
    since jrei has no Rocky/Alma image) to replace the dropped centos7/8
    coverage. rockylinux9 needs a rw cgroup mount rather than the ro mount
    the older jrei images use, since it relies on a real systemd/cgroupv2
    boot rather than the jrei entrypoint's cgroup v1 compatibility shim.
    
    Also add molecule-plugins[docker] to requirements.txt, since molecule's
    docker driver moved out of core into that separate package, and
    gitignore the .ansible/ cache directory ansible-compat creates when
    symlinking the role for local dependency resolution.
    
    refs #17
    
    * fix: stop advertising Fedora/EL as supported platforms
    
    tasks/main.yml only branches on ansible_os_family == "Debian"; there is
    no tasks/redhat.yml or any other os_family branch. Fedora and EL both
    resolve to os_family "RedHat", so the role silently no-ops on them - a
    user targeting Rocky or Fedora would get a "successful" run that
    installs nothing, and Galaxy's metadata would tell them it's supported
    when it isn't.
    
    Drop Fedora and EL from galaxy_info.platforms. The rockylinux9 molecule
    scenario stays as-is: it's CI-only plumbing confirming the role doesn't
    error out on a RedHat-family host, not a functional assertion that
    OpenLDAP gets configured there. Real RedHat/EL task support is a
    separate feature addition, not part of this CI fix.
    
    refs #17
    
    * fix: run molecule scenarios as a parallel matrix, not sequentially
    
    The test step ran all five molecule test invocations in one bash -eo
    pipefail run block, so a failure on the first scenario (debian12)
    aborted the step immediately and produced zero signal on the remaining
    four. Fold scenario into strategy.matrix alongside python-version so
    each scenario runs as its own job and a single failure doesn't hide
    the others. fail-fast: false keeps one red scenario from cancelling
    the rest. This also makes the existing max-parallel: 4 do something -
    previously it sat above a single-value python-version matrix.
    
    refs #17
    
    * fix: work around ansible-core 2.21 breaking molecule docker driver
    
    CI failures on debian12, ubuntu2204, ubuntu2404, and fedora were not a
    community.docker version problem (that was my working hypothesis going
    in, and it was wrong - verified by holding community.docker at 5.2.1
    constant across ansible-core 2.20.7 and 2.21.1 and only the latter
    failed). ansible-core 2.21 stopped attaching `invocation` to registered
    results from looped tasks by default (ansible/ansible#86758,
    "task results" changelog entry). molecule-plugins 25.8.12's docker
    driver create.yml still reads item.invocation.module_args.dest when
    building the scenario's Dockerfile, so every docker-driver scenario
    failed at the create step with:
    
      Finalization of task args for 'community.docker.docker_image' failed:
      Error while resolving value for 'build': object of type 'dict' has no
      attribute 'invocation'
    
    This is a known, already-reported break
    (ansible-community/molecule-plugins#362, #363) with an open, unmerged
    fix (ansible-community/molecule-plugins#364). molecule-plugins 25.8.12
    is already the latest release, so there's nothing to upgrade to yet.
    
    Fix: set ansible.cfg.defaults.inject_invocation: true in each
    docker-driver scenario's molecule.yml, restoring the pre-2.21 behavior
    the create.yml playbook depends on. This has to be set per-scenario via
    molecule's own `ansible.cfg` schema key - molecule always overrides
    ANSIBLE_CONFIG with its own generated config file
    (molecule/provisioner/ansible.py), so a plain repo-root ansible.cfg is
    silently ignored (confirmed by testing - it reproduced the exact same
    failure). Chose this over pinning ansible-core backward since it keeps
    the role on current ansible/molecule versions and stops mattering on
    its own once molecule-plugins#364 ships.
    
    Verified locally in a venv matching CI exactly (ansible==14.1.0,
    ansible-core==2.21.1, molecule==26.6.0, molecule-plugins==25.8.12,
    community.docker==5.2.1): debian12's create step, which previously
    failed with the exact error above, now succeeds. rockylinux9 (no
    Dockerfile build needed, pre_build_image: true) passed full
    dependency/destroy/syntax/create/converge/idempotence/verify/destroy
    with 0 failures under the same ansible-core 2.21.1. debian12's
    converge step still fails locally after create succeeds, but with an
    unrelated "Failed to create temporary directory" error specific to
    running amd64-only jrei images under QEMU emulation on this arm64 Mac
    (jrei/systemd-{debian,ubuntu,fedora} ship no arm64 manifest) - that
    class of failure does not appear anywhere in the actual CI logs, which
    failed at create, and won't occur on GitHub's native amd64 runners.
    
    refs #17
    
    * fix: replace jrei/systemd-* molecule images with mrlesmithjr images
    
    Real CI (GitHub Actions, native amd64 runners) confirmed the
    inject_invocation fix resolved the create-step failure, but debian12,
    ubuntu2204, ubuntu2404, and fedora then failed one step later at
    converge with "Failed to create temporary directory" - a jrei image
    problem (or its interaction with current molecule/ansible-core), not
    an emulation artifact as I'd previously assumed from local-only
    testing.
    
    Rather than keep debugging a third-party image with no clear
    maintenance signal, switch to mrlesmithjr/{debian,ubuntu,fedora,rocky}
    - multi-arch (amd64/arm64/arm) images already proven in production CI
    across ansible-chrony, ansible-dnsmasq, ansible-frr, and
    ansible-guacamole. Platform blocks copied from ansible-frr's
    debian12/ubuntu2204/ubuntu2404/fedora scenarios and
    ansible-dnsmasq's rocky8 scenario (bumped to rocky:9), including their
    proven debian-vs-ubuntu cgroup mount split (ro for debian12; rw +
    cgroupns_mode: host for ubuntu/fedora/rocky) - not a typo, kept as-is
    to match what's actually running successfully elsewhere.
    
    Renamed the rockylinux9 scenario to rocky9 for naming consistency with
    the sibling roles (mrlesmithjr/rocky:9 image, matching their rocky8/
    rocky9 naming convention), and dropped the geerlingguy image's
    pre_build_image: true / empty command workaround since mrlesmithjr/rocky
    uses the same Dockerfile-build flow as the other scenarios.
    
    Pinned mrlesmithjr/fedora to :43 rather than floating - both :42 and
    :43 are current/actively rebuilt on Docker Hub; chose 43 as the more
    recently released of the two rather than defaulting to latest.
    
    Kept ansible.cfg.defaults.inject_invocation: true in every scenario -
    that fix addresses an unrelated ansible-core 2.21 behavior change in
    molecule-plugins' Python code, not anything caused by the jrei images,
    and still applies with these images (verified below).
    
    Because these images are genuinely multi-arch, all 5 scenarios are now
    locally testable on this arm64 Mac (the jrei images were amd64-only
    for every non-EOL tag, which is why 4 of 5 scenarios couldn't be
    locally verified in earlier commits).
    
    refs #17
    
    * fix: point molecule scenarios at the renamed mrlesmithjr images
    
    Completes the previous commit (266aeb6), which only captured the
    rockylinux9 -> rocky9 rename due to a pathspec error on my end - the
    actual image swap and cgroup mount changes described there hadn't
    been staged. This commit is the rest of that change: jrei/systemd-*
    images -> mrlesmithjr/{debian,ubuntu,fedora,rocky}, the ubuntu/fedora/
    rocky9 cgroup mount switch to rw + cgroupns_mode: host, fedora pinned
    to :43, and the workflow matrix entry renamed rockylinux9 -> rocky9.
    
    refs #17
    
    * fix: install debconf-utils and refresh apt cache in tasks/debian.yml
    
    Discovered while getting molecule to actually pass end-to-end against
    current Debian/Ubuntu images, not caused by the CI/image changes
    themselves - this is a pre-existing role bug that CI never exercised
    before because it never got past earlier failures.
    
    Two distinct issues surfaced once converge could actually run:
    
    1. "No package matching 'slapd' is available" - mrlesmithjr/ubuntu and
       mrlesmithjr/debian ship with an empty/stale apt package list cache
       (only 3 entries in /var/lib/apt/lists/, confirmed via apt-cache
       search returning nothing for slapd). The "debian | installing
       packages" task never ran apt-get update, so it depended on whatever
       cache the base image happened to have. Added update_cache: true
       (cache_valid_time: 3600 to avoid updating on every run against a
       real host).
    
    2. Idempotence (second converge run) failed with "Failed to find
       required executable debconf-get-selections". Traced this to
       ansible-core's debconf module: on the vtype: password questions,
       once the debconf question already exists in the database (i.e.
       after slapd's postinst has registered it once), the module calls
       get_password_value(), which requires debconf-get-selections. That
       binary ships in the debconf-utils package, not base debconf (which
       only provides debconf-set-selections/debconf-show - confirmed via
       dpkg -L on both packages). First-run set never hit this path
       because "question not in prev" short-circuits straight to
       set_selection(), which is why this was never caught before. Added a
       task to install debconf-utils before the debconf task runs.
    
    Verified locally: ubuntu2204 and ubuntu2404 (mrlesmithjr images, ci
    ansible-core 2.21.1) now pass converge + idempotence + verify cleanly
    where they previously failed on both counts.
    
    refs #17
    
    * fix: switch debian12 to rw cgroup mount + cgroupns_mode: host
    
    Real CI confirmed the ro-only cgroup mount (copied verbatim from
    ansible-frr's debian12 scenario) fails identically on GitHub Actions'
    native amd64 runners, not just on my local Docker Desktop setup as I'd
    assumed - same "Failed to create temporary directory" /
    UNREACHABLE error in both places. That disproves the theory that ro
    works on GitHub's runners and only breaks locally; it doesn't work for
    this scenario in this repo, period.
    
    Bring debian12 in line with the other 4 scenarios already in this repo
    (ubuntu2204, ubuntu2404, fedora, rocky9), all of which use rw +
    cgroupns_mode: host. This is the exact config I'd already validated
    locally in the previous round but held back to match ansible-frr's
    committed pattern - real CI has now shown that pattern isn't actually
    correct for this repo's debian12 scenario, so applying the fix here.
    
    Why ansible-frr's debian12 gets away with ro is unclear and not
    investigated - possibly an ansible-core version difference, possibly
    something else in that repo's setup. Not chasing it further here since
    it's out of scope for this issue; worth flagging to Larry separately
    as a possible latent issue in ansible-frr's own CI.
    
    Verified locally end-to-end (ansible==14.1.0, ansible-core==2.21.1,
    molecule==26.6.0, molecule-plugins==25.8.12, matching CI): full pass -
    dependency, destroy, syntax, create, converge, idempotence, verify,
    destroy, 0 failures.
    
    refs #17
    
    * fix: drop unused ansible.posix, declare community.docker in requirements.yml
    
    Two MINORs from code review:
    
    1. .github/workflows/default.yml installed ansible.posix alongside
       community.docker, but nothing in tasks/handlers/templates/molecule
       configs uses it (grepped the whole tree, zero hits) - leftover from
       copying the ansible-frr/ansible-dnsmasq workflow pattern. Dropped it.
    
    2. community.docker (needed by molecule-plugins' docker driver) was
       only declared inline in the CI workflow's ansible-galaxy command,
       not in requirements.yml. A contributor running `molecule test`
       locally per the repo's own convention (role-file: requirements.yml)
       without also reading the workflow YAML would be missing it.
    
       Converted requirements.yml from the legacy bare-list roles format to
       the structured roles:/collections: format so both dependency types
       live in one file, and pointed each scenario's dependency.options at
       it for collections too (requirements-file: requirements.yml,
       alongside the existing role-file: requirements.yml) instead of
       leaving molecule to default to a nonexistent per-scenario
       collections.yml. This makes molecule's own dependency stage install
       community.docker automatically, in both CI and local runs, from a
       single declared source - so the standalone
       `ansible-galaxy collection install community.docker ansible.posix`
       line in the workflow is now redundant and removed entirely.
    
    Verified locally (rocky9, ansible==14.1.0/ansible-core==2.21.1 matching
    CI): dependency stage log now shows
    `ansible-galaxy collection install --requirements-file requirements.yml`
    instead of the previous `Missing collections requirements file:
    collections.yml` warning, and the full scenario still passes
    dependency/destroy/syntax/create/converge/idempotence/verify/destroy
    with 0 failures.
    
    refs #17

commit fc5f14b9519f234ff7f6a35f56e8eba44f63ccde
Author: Larry Smith Jr. <mrlesmithjr@gmail.com>
Date:   Sat Mar 28 10:39:02 2026 -0400

    Add grouped dependabot configuration (monthly, pip + actions)

commit 1c97e40d991977b927d9bd290739c6df85834768
Author: Larry Smith Jr <mrlesmithjr@gmail.com>
Date:   Fri Apr 23 13:10:03 2021 -0400

    Create stale.yml

commit cf96be1d47127ea8b7d203cea8614e5068ed0885
Merge: a01aec1 8e8eb04
Author: Larry Smith Jr <mrlesmithjr@gmail.com>
Date:   Tue May 12 01:46:08 2020 -0400

    Merge pull request #12 from em0lar/lint-variable-names
    
    Make ansible-lint more happy with spaces before and after variable names

commit a01aec11f476c2f08e20dec5084bb923ad563ec6
Merge: 50b1fa9 bbafcc9
Author: Larry Smith Jr <mrlesmithjr@gmail.com>
Date:   Tue May 12 01:45:55 2020 -0400

    Merge pull request #11 from UdelaRInterior/managing-overlays
    
    Managing Overlays

commit 8e8eb0491e78e08103859b2b05cacc18506ba405
Author: Leo Maroni <git@em0lar.de>
Date:   Sat May 9 01:45:45 2020 +0200

    Make ansible-lint more happy with spaces before and after variable names

commit bbafcc992d5c334dff8ee44dd004075009e0bd45
Author: Andrés <apias@cci.edu.uy>
Date:   Wed Mar 18 00:33:24 2020 -0300

    Managing Overlays
    
    The necesary ldif files to add and config your overlays or acls must be pleaced in files/openldap/overlays/

commit 50b1fa991a4f5392aa58e2cb72bda2812d9aedf3
Merge: b4a425f a87119e
Author: Larry Smith Jr <mrlesmithjr@gmail.com>
Date:   Wed Apr 1 13:48:16 2020 -0400

    Merge pull request #10 from UdelaRInterior/extending-schema
    
    Extending LDAP schema

commit a87119e942917624a81154217c99e599b2f8da1e
Author: Andrés <apias@cci.edu.uy>
Date:   Mon Mar 9 16:17:16 2020 -0300

    Extending LDAP schema
    
    For a non basic schema, Its .schema file must be placed in files/openldap/schema/. This file will be copied and converted to ldif before being inserted to LDAP tree

commit b4a425fa8852bfa27a8ec530702f224180df7971
Merge: b011f9a ae81b86
Author: Larry Smith Jr <mrlesmithjr@gmail.com>
Date:   Mon Mar 9 14:32:56 2020 -0400

    Merge pull request #9 from UdelaRInterior/phpldapadmin-optional-config
    
    Making phpldapadmin config optional

commit ae81b86f85981ba9a230d8eceb76c67fc0b1d181
Author: Andrés <apias@cci.edu.uy>
Date:   Tue Mar 3 18:37:53 2020 -0300

    Making phpldapadmin config optional
    That is, if ldapadmin is not installed, it should not be configured
    Also, phpldapadmin is not available in Debian Buster repositories

commit b011f9a6a00910062e752413aa14b8a6865f89ff
Merge: f313ef3 9780033
Author: Larry Smith Jr <mrlesmithjr@gmail.com>
Date:   Thu Feb 27 20:09:07 2020 -0500

    Merge pull request #8 from mrlesmithjr/refactoring
    
    Refactoring

commit 978003343ed7f04bb0a7fe229330aff2606c5c3e
Author: Larry Smith Jr <mrlesmithjr@gmail.com>
Date:   Thu Feb 27 20:00:30 2020 -0500

    Updated changelog

commit 9a1445b592d07cff4cf75085dd9cb61b01ed9ee8
Author: Larry Smith Jr <mrlesmithjr@gmail.com>
Date:   Thu Feb 27 20:00:19 2020 -0500

    Resolving idempotency issue
    
    - Added changed_when false for debconf settings.
    - These settings should not be seen as a change.

commit 9cd47212e5b3da367aa906d549942613b9712093
Author: Larry Smith Jr <mrlesmithjr@gmail.com>
Date:   Thu Feb 27 19:51:11 2020 -0500

    Updated changelog

commit eab1855654d8429988fa916a4b2d334335c1d901
Author: Larry Smith Jr <mrlesmithjr@gmail.com>
Date:   Thu Feb 27 19:51:02 2020 -0500

    Disabled ansible-etc-hosts role
    
    - Not needed for now plus causes issues with testing in containers

commit 6d6ed4cab029cd6e7e704e21f4693f58250a2eb7
Author: Larry Smith Jr <mrlesmithjr@gmail.com>
Date:   Thu Feb 27 18:38:33 2020 -0500

    Updated changelog

commit 8419056f7c519fc81b63031ef59d942a34e26511
Author: Larry Smith Jr <mrlesmithjr@gmail.com>
Date:   Thu Feb 27 18:38:24 2020 -0500

    Fixing Ansible linting

commit 5276cf3dd8bc58de197c54c387b8bd8420286892
Author: Larry Smith Jr <mrlesmithjr@gmail.com>
Date:   Thu Feb 27 18:31:10 2020 -0500

    Updated CHANGELOG

commit 942511e1168d42c41ad7b4172371b9c104263e9d
Author: Larry Smith Jr <mrlesmithjr@gmail.com>
Date:   Thu Feb 27 18:30:51 2020 -0500

    Updated playbook format

commit 3f0c1680960b89f22c0a4c3b2b911e27b3f27745
Author: Larry Smith Jr <mrlesmithjr@gmail.com>
Date:   Thu Feb 27 18:30:35 2020 -0500

    Cleaned up formatting, etc.

commit 61ef009a630f0a68a1fe082b47fd513cc48cdaa8
Author: Larry Smith Jr <mrlesmithjr@gmail.com>
Date:   Thu Feb 27 18:30:02 2020 -0500

    Disabled all tests except Ubuntu

commit c170a264255e48ee65ec5d5a0acaae9c2ce88f75
Author: Larry Smith Jr <mrlesmithjr@gmail.com>
Date:   Thu Feb 27 18:05:41 2020 -0500

    Added new Molecule tests, etc. from new structure

commit 2e3e48657c8b6bc88c1d9751e41b16d6bddb4fb1
Author: Larry Smith Jr <mrlesmithjr@gmail.com>
Date:   Thu Feb 27 18:05:27 2020 -0500

    Updated files, etc. after new structure

commit 803484ba8d7357ba5778bdf03cd7be211cb23027
Author: Larry Smith Jr <mrlesmithjr@gmail.com>
Date:   Thu Feb 27 17:56:26 2020 -0500

    Deleted old tests, etc. not needed

commit f313ef339892360137f5d38bcb34302ce61e6844
Merge: 05da717 70ab6ec
Author: Larry Smith Jr <mrlesmithjr@gmail.com>
Date:   Fri Nov 16 01:15:55 2018 -0500

    Merge pull request #6 from LPottmeier/master
    
    Fix missing php-xml issue

commit 70ab6ecb9dd5101189154f57453de23b86d99718
Author: Ludger Pottmeier <ludger.pottmeier@atos.net>
Date:   Thu Nov 15 15:52:59 2018 +0100

    Fix missing php-xml issue

commit 05da71701f27cf9ed2ee310673433295cc474ac9
Merge: 6b3eb20 8ef138d
Author: Larry Smith Jr <mrlesmithjr@gmail.com>
Date:   Fri Sep 28 09:29:39 2018 -0400

    Merge pull request #4 from arunderwood/travis_multiple_os_testing
    
    Testing multiple OS via Travis

commit 6b3eb2023f25483edd52e8cbe5971204306b6fd2
Merge: bf72c2e 3205ed1
Author: Larry Smith Jr <mrlesmithjr@gmail.com>
Date:   Fri Sep 28 09:28:08 2018 -0400

    Merge pull request #3 from arunderwood/fixes
    
    Use 'service' module instead of 'systemd' for greater compatibility

commit 8ef138de7b6994bc0a8f277a843bab0bcfec3fc4
Author: Andrew Underwood <andrew.underwood@workiva.com>
Date:   Thu Sep 27 22:58:31 2018 -0500

    Configure travis to run tests on:
        * Ubuntu Trusty
        * Ubuntu Xenial
        * Ubuntu Bionic

commit 3205ed11684b02ad17a76ea5ea618a4dd9aea181
Author: James Kang <marchkang@gmail.com>
Date:   Sat Aug 4 23:39:22 2018 -0700

    getting Ubuntu 14.4 to work around the issues of OS provided pip not
    being able to install "ansible" and quick/dirty fix on using 'service'
    instead of 'systemd'

commit bf72c2e9dcc4be7d903ed15df039da92f51dde6e
Merge: e6dfad9 30c3046
Author: Larry Smith Jr <mrlesmithjr@gmail.com>
Date:   Mon Feb 12 20:18:06 2018 -0500

    Merge pull request #1 from bunchc/cleanup
    
    Added or updated to work on ubuntu 16.04

commit 30c30465b6c8af976b6775d683808d1ca1e57c86
Author: bunchc <bunchc@gmail.com>
Date:   Mon Feb 12 15:28:29 2018 -0600

    defaults/main.yml - Broke out domain components for easier reuse
    tasks/debian.yml - Populating openLDAP task was failing due to permissions against the nodomain org. Added  slapd/domain and shared/organization to debconf to correct this
    tasks/debian.yml - with_items clause needed {{}} to not error
    tasks/config_openldap.yml - Added task to set admin password
    tasls/config_openldap.yml - Added task to restart slapd

commit e6dfad98aa05d2ab60358afaad6d78daa45111bd
Author: Larry Smith Jr <mrlesmithjr@gmail.coml>
Date:   Tue Feb 16 17:02:35 2016 -0500

    Updated role info
    
    Signed-off-by: Larry Smith Jr <mrlesmithjr@gmail.coml>

commit b5e4c8269b0f83375ec8a60bc6f6496e0a1bde63
Author: Larry Smith Jr <mrlesmithjr@gmail.coml>
Date:   Tue Feb 16 17:01:52 2016 -0500

    Updated final tasks
    
    Signed-off-by: Larry Smith Jr <mrlesmithjr@gmail.coml>

commit 190060cf333ef08c6928803e67d2cc0a09274884
Author: Larry Smith Jr <mrlesmithjr@gmail.coml>
Date:   Tue Feb 16 16:35:48 2016 -0500

    Updated tasks, created add'l vars and etc.
    
    Signed-off-by: Larry Smith Jr <mrlesmithjr@gmail.coml>

commit 2b02f1f28a79082b70ca5713aa1284da737a33ec
Author: Larry Smith Jr <mrlesmithjr@gmail.coml>
Date:   Tue Feb 16 15:39:12 2016 -0500

    Added add'l vars and configurations for install
    
    Signed-off-by: Larry Smith Jr <mrlesmithjr@gmail.coml>

commit 75cc36e11fe434b0836b8979e6e0fecba234ac2c
Author: Larry Smith Jr <mrlesmithjr@gmail.coml>
Date:   Tue Feb 16 15:01:39 2016 -0500

    Added ldap and phpldapadmin configs
    
    Signed-off-by: Larry Smith Jr <mrlesmithjr@gmail.coml>

commit 5b82c14fbd3172831e8369e36354e3c1adb295fb
Author: Larry Smith Jr <mrlesmithjr@gmail.coml>
Date:   Tue Feb 16 13:47:54 2016 -0500

    first commit
