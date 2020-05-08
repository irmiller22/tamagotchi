# Python - Circular Imports

## What is a circular import?

Every Python engineer will run into the issues of circular imports at some point in their career. A circular import arises whenever there is an interdependence between different Python modules. Let's illustrate this issue with an example.

### Code example

```py
# dialog.py

import app


class Dialog:
    def __init__(self, save_directory):
        self.save_directory = save_directory

    ...

save_dialog = Dialog(app.prefs.get('save_dir'))

def show(self):
    ...
```

The problem is that the `app` module that contains the `prefs` object also imports the `Dialog` class in order to show the same dialog on program start:

```py
# app.py

import dialog


class Prefs:
    ...
    def get(self, name):
        ...

prefs = Prefs()
dialog.show()
```

When I try to run the program, the following exception occurs:

```bash
$ python3 main.py
Traceback (most recent call last):
  File ".../main.py", line 17, in <module>
    import app
  File ".../app.py", line 17, in <module>
    import dialog
  File ".../dialog.py", line 23, in <module>
    save_dialog = Dialog(app.prefs.get('save_dir'))
AttributeError: partially initialized module 'app' has no
➥ attribute 'prefs' (most likely due to a circular import)
```

This occurs due to a collision somewhere in the Python import heirarchy. We'll dig into this a little bit more in the next section.

### How does Python handle imports?

Python relies on an internal built-in package called [`importlib`](https://docs.python.org/3/library/importlib.html) in order to handle imports. If you're not familiar with the package and its internals, here's what it does at a high level during an import:

1. Searches for a Python module in locations from `sys.path`
2. Loads the code from a module and ensures that it compiles
3. Creates a corresponding empty module object
4. Inserts the module into `sys.modules`
5. Runs the code in the module object to define its contents

So how does this lead to a circular import error?

### How does a circular import occur?

Typically, a circular import occurs after step 4 in the previous section. The attributes in the code aren't really defined until the code has executed. So how can we break this down further with the code example?

In the example, the `app` module imports the `dialog` module before defining anything. Then the `dialog` module imports `app`. Since the `app` module isn't finished running, the `app` module is empty. As a result, the `AttributeError` exception gets raised because the code that defines `prefs` hasn't run yet.

## How to fix circular imports?

In order to fix a circular import, the recommended approach is to refactor the code so that the problematic import (in this case, the `prefs` code) is defined at the bottom of the dependency tree. Then both the `app` and `dialog` modules can import the same utility module and avoid a circular dependency.

### Refactor and reorder Dependency Tree

See below for an example for resolving the circular import in the example given above:

```py
# utils.py
class Prefs:
    ...
    def get(self, name):
        ...

prefs = Prefs()
```

```py
# dialog.py

import app
import utils


class Dialog:
    def __init__(self, save_directory):
        self.save_directory = save_directory

    ...

save_dialog = Dialog(app.prefs.get('save_dir'))

def show(self):
    ...
```

```py
# app.py

import dialog
import utils


utils.prefs
dialog.show()
```

By moving the `prefs` code to the `utils` module, we're able to resovle the circular import, because we've now moved the `prefs` code to the bottom of hte dependency tree. However, it isn't always possible to to implement such a clear division in logic. There are several other ways we can handle circular imports: reordering imports, minimize side effects of imports at runtime, and dynamic imports. Keep in mind that the following fixes are not considered "clean" fixes.

### Reordering imports

The first approach is to change the order of the imports.

```py
# app.py
class Prefs:
    ...

prefs = Prefs()

import dialog # moved
dialog.show()
```

This addresses the circular import because by the time `dialog` is imported, `prefs` has already been loaded in the `app` module. However, this goes against the PEP8 style guide, which recommends that you always put your imports at the top of the file. This is also a brittle solution because it can break your code.

### Minimize import side effects at runtime

The second approach is to minimize side effects of imports at runtime. What does this mean? This means that instead of loading and running functions at import time, we actually avoid doing so. Each module would have a setup function defined (in our example, we'll use the `configure` function as our setup function). We'd call the setup function once everything has been imported. See below for this approach in action:

```py
# dialog.py

import app

class Dialog:
    ...

save_dialog = Dialog()

def show():
    ...

def configure():
    save_dialog.save_dir = app.prefs.get("save_dir")
```

```py
# app.py

import dialog

class Prefs:
    ...

prefs = Prefs()

def configure():
    ...
```

Then finally, in the `main` module, we define three phases of execution: import everything, `configure` everything, and run the first line of code.

```py
# main.py

import app
import dialog

app.configure()
dialog.configure()

dialog.show()
```

This tends to work well in most cases, but it can still be difficult to structure your modules to have an explicit `configure` function. This also makes the code a little more difficult to read because it separates the code definition from the code configuration.

### Dynamic imports

Finally, we have dynamic imports. This is the simplest approach, since it just involves importing a module within a method so the module gets loaded when the method is called. It's called a _dynamic import_ because the import happens while the code is running, not while the code is starting up and initializing the modules.

See below for an example:

```py
# dialog.py
class Dialog:
    ...

save_dialog = Dialog()

def show():
    import app
    save_dialog.save_dir = app.prefs.get("save_dir")
    ...
```

Nothing would change in the `app` module:

```py
# app.py

import dialog

class Prefs:
    ...

prefs = Prefs()
dialog.show()
```

It's a good practice to avoid dynamic imports unless it's absolutely necessary. By delaying execution of the import statement until runtime, you can also experience unexpected failures after the code has initialized.
