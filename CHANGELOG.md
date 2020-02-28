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

commit 70ab6ecb9dd5101189154f57453de23b86d99718
Author: Ludger Pottmeier <ludger.pottmeier@atos.net>
Date:   Thu Nov 15 15:52:59 2018 +0100

    Fix missing php-xml issue

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
