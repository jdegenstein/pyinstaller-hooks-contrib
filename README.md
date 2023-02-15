# `pyinstaller-hooks-contrib`: The PyInstaller community hooks repository

What happens when (your?) package doesn't work with PyInstaller? Say you have data files that you need at runtime?
PyInstaller doesn't bundle those. Your package requires others which PyInstaller can't see? How do you fix that?

In summary, a "hook" file extends PyInstaller to adapt it to the special needs and methods used by a Python package.
The word "hook" is used for two kinds of files. A runtime hook helps the bootloader to launch an app, setting up the
environment. A package hook (there are several types of those) tells PyInstaller what to include in the final app -
such as the data files and (hidden) imports mentioned above.

This repository is a collection of hooks for many packages, and allows PyInstaller to work with these packages
seamlessly.


## Installation

`pyinstaller-hooks-contrib` is automatically installed when you install PyInstaller, or can be installed with pip:

```commandline
pip install -U pyinstaller-hooks-contrib
```


## I can't see a hook for `a-package`

Either `a-package` works fine without a hook, or no-one has contributed hooks.
If you'd like to add a hook, or view information about hooks,
please see below.


## Hook configuration (options)

Hooks that support configuration (options) and their options are documented in
[Supported hooks and options](hooks-config.rst).


## I want to help!

If you've got a hook you want to share then great!
The rest of this page will walk you through the process of contributing a hook.
If you've been here before then you may want to skip to the [summary checklist](#summary)

**Unless you are very comfortable with `git rebase -i`, please provide one hook per pull request!**
**If you have more than one then submit them in separate pull requests.**


### Setup

[Fork this repo](https://github.com/pyinstaller/pyinstaller-hooks-contrib/fork) if you haven't already done so.
(If you have a fork already but its old, click the **Fetch upstream** button on your fork's homepage.)
Clone and `cd` inside your fork by running the following (replacing `bob-the-barnacle` with your github username):

```
git clone https://github.com/bob-the-barnacle/pyinstaller-hoooks-contrib.git
cd pyinstaller-hooks-contrib
```

Create a new branch for you changes (replacing `foo` with the name of the package):
You can name this branch whatever you like.

```
git checkout -b hook-for-foo
```

If you wish to create a virtual environment then do it now before proceeding to the next step.

Install this repo in editable mode.
This will overwrite your current installation.
(Note that you can reverse this with `pip install --force-reinstall pyinstaller-hooks-contrib`).

```
pip install -e .
pip install -r requirements-test.txt
pip install flake8
```

Note that on macOS and Linux, `pip` may by called `pip3`.
If you normally use `pip3` and `python3` then use `pip3` here too.
You may skip the 2<sup>nd</sup> line if you have no intention of providing tests (but please do provide tests!).


### Add the hook

Standard hooks live in the [src/_pyinstaller_hooks_contrib/hooks/stdhooks/](../master/src/_pyinstaller_hooks_contrib/hooks/stdhooks/) directory.
Runtime hooks live in the [src/_pyinstaller_hooks_contrib/hooks/rthooks/](../master/src/_pyinstaller_hooks_contrib/hooks/rthooks/) directory.
Simply copy your hook into there.
If you're unsure if your hook is a runtime hook then it almost certainly is a standard hook.

Please annotate (with comments) anything unusual in the hook.
*Unusual* here is defined as any of the following:

*   Long lists of `hiddenimport` submodules.
    If you need lots of hidden imports then use [`collect_submodules('foo')`](https://pyinstaller.readthedocs.io/en/latest/hooks.html#PyInstaller.utils.hooks.collect_submodules).
    For bonus points, track down why so many submodules are hidden. Typical causes are:
    *   Lazily loaded submodules (`importlib.importmodule()` inside a module `__getattr__()`).
    *   Dynamically loaded *backends*.
    *   Usage of `Cython` or Python extension modules containing `import` statements.
*   Use of [`collect_all()`](https://pyinstaller.readthedocs.io/en/latest/hooks.html#PyInstaller.utils.hooks.collect_all).
    This function's performance is abismal and [it is broken by
    design](https://github.com/pyinstaller/pyinstaller/issues/6458#issuecomment-1000481631) because it confuses
    packages with distributions.
    Check that you really do need to collect all of submodules, data files, binaries, metadata and dependencies.
    If you do then add a comment to say so (and if you know it - why).
    Do not simply use `collect_all()` just to *future proof* the hook.
*   Any complicated `os.path` arithmetic (by which I simply mean overly complex filename manipulations).


#### Add the copyright header

All source files must contain the copyright header to be covered by our terms and conditions.

If you are **adding** a new hook (or any new python file), copy/paste the appropriate copyright header (below) at the top
replacing 2021 with the current year.

<details><summary>GPL 2 header for standard hooks or other Python files.</summary>

```python
# ------------------------------------------------------------------
# Copyright (c) 2021 PyInstaller Development Team.
#
# This file is distributed under the terms of the GNU General Public
# License (version 2.0 or later).
#
# The full license is available in LICENSE.GPL.txt, distributed with
# this software.
#
# SPDX-License-Identifier: GPL-2.0-or-later
# ------------------------------------------------------------------
```

</details>

<details><summary>APL header for runtime hooks only.
Again, if you're unsure if your hook is a runtime hook then it'll be a standard hook.</summary>

```python
# ------------------------------------------------------------------
# Copyright (c) 2021 PyInstaller Development Team.
#
# This file is distributed under the terms of the Apache License 2.0
#
# The full license is available in LICENSE.APL.txt, distributed with
# this software.
#
# SPDX-License-Identifier: Apache-2.0
# ------------------------------------------------------------------
```

</details>


If you are **updating** a hook, skip this step.
Do not update the year of the copyright header - even if it's out of date.


### Test

Having tests is key to our continuous integration.
With them we can automatically verify that your hook works on all platforms, all Python versions and new versions of
libraries as and when they are released.
Without them, we have no idea if the hook is broken until someone finds out the hard way.
Please write tests!!!

Some user interface libraries may be impossible to test without user interaction
or a wrapper library for some web API may require credentials (and possibly a paid subscription) to test.
In such cases, don't provide a test.
Instead explain either in the commit message or when you open your pull request why an automatic test is impractical
then skip on to [the next step](#run-linter).


#### Write tests(s)

A test should be the least amount of code required to cause a breakage
if you do not have the hook which you are contributing.
For example if you are writing a hook for a library called `foo`
which crashes immediately under PyInstaller on `import foo` then `import foo` is your test.
If `import foo` works even without the hook then you will have to get a bit more creative.
Good sources of such minimal tests are introductory examples
from the documentation of whichever library you're writing a hook for.
Package's internal data files and hidden dependencies are prone to moving around so
tests should not explicitly check for presence of data files or hidden modules directly -
rather they should use parts of the library which are expected to use said data files or hidden modules.

Tests currently all live in [src/_pyinstaller_hooks_contrib/tests/test_libraries.py](../master/src/_pyinstaller_hooks_contrib/tests/test_libraries.py).
Navigate there and add something like the following, replacing all occurrences of `foo` with the real name of the library.
(Note where you put it in that file doesn't matter.)

```python
@importorskip('foo')
def test_foo(pyi_builder):
    pyi_builder.test_source("""

        # Your test here!
        import foo

        foo.something_fooey()

    """)
```

If the library has changed significantly over past versions then you may need to add version constraints to the test.
To do that, replace the `@importorskip("foo")` with a call to `PyInstaller.utils.tests.requires()` (e.g.
`@requires("foo >= 1.4")`) to only run the test if the given version constraint is satisfied.
Note that `@importorskip` uses module names (something you'd `import`) whereas `@requires` uses distribution names
(something you'd `pip install`) so you'd use `@importorskip("PIL")` but `@requires("pillow")`.
For most packages, the distribution and packages names are the same.


#### Run the test locally

Running our full test suite is not recommended as it will spend a very long time testing code which you have not touched.
Instead, run tests individually using either the `-k` option to search for test names:

```
pytest -k test_foo
```

Or using full paths:
```
pytest src/_pyinstaller_hooks_contrib/tests/test_libraries.py::test_foo
```


#### Pin the test requirement

Get the version of the package you are working with (`pip show foo`)
and add it to the [requirements-test-libraries.txt](../master/requirements-test-libraries.txt) file.
The requirements already in there should guide you on the syntax.


#### Run the test on CI/CD

To test hooks on all platforms we use Github's continuous integration (CI/CD).
Our CI/CD is a bit unusual in that it's triggered manually and takes arguments
which limit which tests are run.
This is for the same reason we filter tests when running locally -
the full test suite takes ages.

First push the changes you've made so far.

```commandline
git push --set-upstream origin hook-for-foo
```

Replace *billy-the-buffalo* with your Github username in the following url then open it.
It should take you to the `oneshot-test` actions workflow on your fork.
You may be asked if you want to enable actions on your fork - say yes.
```
https://github.com/billy-the-buffalo/pyinstaller-hooks-contrib/actions/workflows/oneshot-test.yml
```

Find the **Run workflow** button and click on it.
If you can't see the button,
select the **Oneshot test** tab from the list of workflows on the left of the page
and it should appear.
A dialog should appear containing one drop-down menu and 5 line-edit fields.
This dialog is where you specify what to test and which platforms and Python versions to test on.
Its fields are as follows:

1.  A branch to run from. Set this to the branch which you are using (e.g. ``hook-for-foo``),
2.  Which package(s) to install and their version(s).
    Which packages to test are inferred from which packages are installed.
    You can generally just copy your own changes to the `requirements-test-libraries.txt` file into this box.
    * Set to `foo` to test the latest version of `foo`,
    * Set to `foo==1.2, foo==2.3` (note the comma) to test two different versions of `foo` in separate jobs,
    * Set to `foo bar` (note the lack of a comma) to test `foo` and `bar` in the same job,
3.  Which OS or OSs to run on
    * Set to `ubuntu` to test only `ubuntu`,
    * Set to `ubuntu, macos, windows` (order is unimportant) to test all three OSs.
4.  Which Python version(s) to run on
    * Set to `3.9` to test only Python 3.9,
    * Set to `3.8, 3.9, 3.10, 3.11` to test all currently supported version of Python.
5.  The final two options can generally be left alone.

Hit the green **Run workflow** button at the bottom of the dialog, wait a few seconds then refresh the page.
Your workflow run should appear.

We'll eventually want to see a build (or collection of builds) which pass on
all OSs and all Python versions.
Once you have one, hang onto its URL - you'll need it when you submit the pull request.
If you can't get it to work - that's fine.
Open a pull request as a draft, show us what you've got and we'll try and help.


#### Triggering CI/CD from a terminal

If you find repeatedly entering the configuration into Github's **Run workflow** dialog arduous
then we also have a CLI script to launch it.
Run ``python scripts/cloud-test.py --help`` which should walk you through it.
You will have to enter all the details again but, thanks to the wonders of terminal history,
rerunning a configuration is just a case of pressing up then enter.


### Run Linter

We use `flake8` to enforce code-style.
`pip install flake8` if you haven't already then run it with the following.
Note that this assumes that you did create a new branch in the [setup step](#setup).

```
git diff -U0 master | flake8 --diff -
```

No news is good news.
If it complains about your changes then do what it asks then run it again.
If you don't understand the errors it come up with them lookup the error code
in each line (a capital letter followed by a number e.g. `W391`).

If it complains about code which you haven't written,
or if you didn't create a new branch at the start then, using `git log`,
find the commit ID of the newest commit which you didn't write, copy it
and replace `master` in the above command with that ID.

```
git diff -U0 a5d3841c282fa23fd68c3d6a85519e73c08acb4a | flake8 --diff -
```

**Please do not fix flake8 issues found in parts of the repository other than the bit that you are working on.** Not only is it very boring for you, but it is harder for maintainers to
review your changes because so many of them are irrelevant to the hook you are adding or changing.


### Add a news entry

Please read [news/README.txt](https://github.com/pyinstaller/pyinstaller-hooks-contrib/blob/master/news/README.txt) before submitting you pull request.
This will require you to know the pull request number before you make the pull request.
You can usually guess it by adding 1 to the number of [the latest issue or pull request](https://github.com/pyinstaller/pyinstaller-hooks-contrib/issues?q=sort%3Acreated-desc).
Alternatively, [submit the pull request](#submit-the-pull-request) as a draft,
then add, commit and push the news item after you know your pull request number.


### Summary

A brief checklist for before submitting your pull request:

* [ ] All new Python files have [the appropriate copyright header](#add-the-copyright-header).
* [ ] You have written a [news entry](#add-a-news-entry).
* [ ] Your changes [satisfy the linter](#run-linter) (run `git diff -U0 master | flake8 --diff -`).
* [ ] You have written tests (if possible), [pinned the test requirement](#pin-the-test-requirement) and linked to a successful CI build.


### Submit the pull request

Once you've done all the above, go ahead and create a pull request.
If you're stuck doing any of the above steps, create a draft pull request and explain what's wrong - we'll sort you out...
Feel free to copy/paste commit messages into the Github pull request title and description.
If you have run CI/CD, please include a link to it in your description so that we can see that it works.
If you've never done a pull request before, note that you can edit it simply by running `git push` again.
No need to close the old one and start a new one.

---

If you plan to contribute frequently or are interested in becoming a developer,
send an email to `legorooj@protonmail.com` to let us know.
