# USoC21 Hackathon

## [4. Run Django on Unikraft](https://usoc21.unikraft.org/docs/hackathon/#4-run-django-on-unikraft)

### Prepare

```
kraft configure
kraft menuconfig # Library Configuration -> vfscore -> Default root device -> fs0
kraft build
```

The filesystem:
```
mkdir python_fs && cd $_
cp ../minrootfs.tgz .
tar -xf minrootfs.tgz
```

### Run

Interactive mode:
```
qemu-guest -k build/app-django_kvm-x86_64 -m 256 -e python_fs/ -a ""
```

[Django server](
https://docs.djangoproject.com/en/3.2/intro/tutorial01/):
```
qemu-guest -k build/app-django_kvm-x86_64 -m 256 -e python_fs/ -a "mysite/manage.py runserver"
```

### Requirements:
```
install_requires = 
	asgiref >= 3.3.2, < 4
	pytz
	sqlparse >= 0.2.2
```

Asgiref:
```
install_requires = 
	typing_extensions; python_version < "3.8"
```

Installed packages:
- Django-3.2.7
- sqlparse-0.4.1
- pytz-2021.1
- asgiref-3.4.1
- typing_extensions-3.10.0.2 - not yet

The packages are added in `python_fs/lib/python3.7/site-packages/`.

### Status

At the moment, restarting the server is disabled (`python_fs/lib/python3.7/site-packages/django/core/management/commands/runserver.py`):
```
    def run(self, **options):
        """Run the server, using the autoreloader if needed."""
        # use_reloader = options['use_reloader']

        # if use_reloader:
        #     autoreload.run_with_reloader(self.inner_run, **options)
        # else:
        #     self.inner_run(None, **options)
        self.inner_run(None, **options)
```

Got the error:
```
  File "/lib/python3.7/sqlite3/__init__.py", line 23, in <module>
    from sqlite3.dbapi2 import *
  File "/lib/python3.7/sqlite3/dbapi2.py", line 27, in <module>
    from _sqlite3 import *
ModuleNotFoundError: No module named '_sqlite3'
```
when not enabling the Python3 Sqlite extenson, so I also added the sqlite
library and enabled the extension:
```
adina@asm:app-django$ cat .config | grep SQLITE
CONFIG_LIBPYTHON3_EXTENSION_SQLITE=y
CONFIG_LIBSQLITE=y
# CONFIG_LIBSQLITE_MAIN_FUNCTION is not set
```

Added `extern PyObject* PyInit__sqlite3(void);` in `libs/python3/modules_config.c` in order to fix:
```
/media/Adina/ACS/Unikraft/usoc/hackathon/libs/python3/modules_config.c:293:16: error: ‘PyInit__sqlite3’ undeclared here (not in a function); did you mean ‘PyInit__tkinter’?
  293 |     {"sqlite", PyInit__sqlite3},
      |                ^~~~~~~~~~~~~~~
      |                PyInit__tkinter
/media/Adina/ACS/Unikraft/usoc/hackathon/libs/python3/modules_config.c:293:5: warning: missing initializer for field ‘initfunc’ of ‘struct _inittab’ [-Wmissing-field-initializers]
  293 |     {"sqlite", PyInit__sqlite3},
      |     ^
In file included from /media/Adina/ACS/Unikraft/usoc/hackathon/apps/app-django/build/libpython3/origin/Python-3.7.4/Include/Python.h:145,
                 from /media/Adina/ACS/Unikraft/usoc/hackathon/libs/python3/include/Python.h:41,
                 from /media/Adina/ACS/Unikraft/usoc/hackathon/libs/python3/modules_config.c:20:
/media/Adina/ACS/Unikraft/usoc/hackathon/apps/app-django/build/libpython3/origin/Python-3.7.4/Include/import.h:122:17: note: ‘initfunc’ declared here
  122 |     PyObject* (*initfunc)(void);
      |                 ^~~~~~~~
```
and got this error:
```
/media/Adina/ACS/Unikraft/usoc/hackathon/apps/app-django/build/libpython3/origin/Python-3.7.4/Modules/_sqlite/cache.c:260:9: error: ‘MODULE_NAME’ undeclared here (not in a function)
  260 |         MODULE_NAME "Node",                             /* tp_name */
      |         ^~~~~~~~~~~
/media/Adina/ACS/Unikraft/usoc/hackathon/apps/app-django/build/libpython3/origin/Python-3.7.4/Modules/_sqlite/cache.c:260:21: error: expected ‘}’ before string constant
  260 |         MODULE_NAME "Node",                             /* tp_name */
      |                     ^~~~~~
/media/Adina/ACS/Unikraft/usoc/hackathon/apps/app-django/build/libpython3/origin/Python-3.7.4/Modules/_sqlite/cache.c:258:34: note: to match this ‘{’
  258 | PyTypeObject pysqlite_NodeType = {
      |                                  ^
/media/Adina/ACS/Unikraft/usoc/hackathon/apps/app-django/build/libpython3/origin/Python-3.7.4/Modules/_sqlite/cache.c:302:21: error: expected ‘}’ before string constant
  302 |         MODULE_NAME ".Cache",                           /* tp_name */
      |                     ^~~~~~~~
/media/Adina/ACS/Unikraft/usoc/hackathon/apps/app-django/build/libpython3/origin/Python-3.7.4/Modules/_sqlite/cache.c:300:35: note: to match this ‘{’
  300 | PyTypeObject pysqlite_CacheType = {
```

Added to `build/libpython3/origin/Python-3.7.4/Modules/_sqlite/cache.c`:
```
#ifndef MODULE_NAME
#define MODULE_NAME "sqlite"
#endif
```
and got:
```
In file included from /media/Adina/ACS/Unikraft/usoc/hackathon/apps/app-django/build/libpython3/origin/Python-3.7.4/Modules/_sqlite/connection.c:27:
/media/Adina/ACS/Unikraft/usoc/hackathon/apps/app-django/build/libpython3/origin/Python-3.7.4/Modules/_sqlite/connection.h:33:10: fatal error: sqlite3.h: No such file or directory
   33 | #include "sqlite3.h"
      |          ^~~~~~~~~~~
compilation terminated.
```

Copied the missing header:
```
cp build/libsqlite/origin/sqlite-amalgamation-3300100/sqlite3.h ../../libs/sqlite/include/
```.
