---
layout: post
title: Continuous integration for appengine
subtitle: Setup a continuous integration / delivery pipeline on drone.io for google app engine
---

I got different errors trying to setup the build script on [drone.io](http://drone.io) on early attempts. Firstly it is necessary to provide a requirements.txt file listing dependencies you use in your application.
I was running unittests and the mock module was not available.


Some moule install script uses setuptools, so this line was also necessary

```bash
pip install --upgrade setuptools
```

Next I had to figure out a way to include libraries from the google app engine sdk. I found a convenient script that downloads and installs the latest version.

I kept getting import error for gae sdk, namely webapp2, webob, ...

# Check the PYTHONPATH

Even thought I downloaded the gae_sdk and correctly setup the PYTHONPATH environment variable I kept getting errors.
To diagnose the issue I printed both the PYTHONPATH env and the sys.path from python.
Both tests where positive, the google_appengine path was there... What happens?

```bash
echo $PYTHONPATH
python -c "import sys; print(sys.path)"
```

# Set ENVIRONMENT variables on drone.io

The build task configuration on [drone.io](http://drone.io) has a field where environment variables can be listed, I tried to replicate there the PYTHONPATH configuration.

Tested the build for the 8th time and no luck! the path was listed twice obviously without the desired effect.

# Devappserver2 test suit organization

Looking inside the google_appengine/run_tests.py script I noticed something interesting:

```python
DIR_PATH = os.path.dirname(__file__)

TEST_LIBRARY_PATHS = [
    DIR_PATH,
    os.path.join(DIR_PATH, 'lib', 'cherrypy'),
    os.path.join(DIR_PATH, 'lib', 'fancy_urllib'),
    os.path.join(DIR_PATH, 'lib', 'yaml-3.10'),
    os.path.join(DIR_PATH, 'lib', 'antlr3'),
    os.path.join(DIR_PATH, 'lib', 'concurrent'),
    os.path.join(DIR_PATH, 'lib', 'ipaddr'),
    os.path.join(DIR_PATH, 'lib', 'jinja2-2.6'),
    os.path.join(DIR_PATH, 'lib', 'webob-1.2.3'),
    os.path.join(DIR_PATH, 'lib', 'webapp2-2.5.1'),
    os.path.join(DIR_PATH, 'lib', 'mox'),
    os.path.join(DIR_PATH, 'lib', 'protorpc-1.0'),
]


def main():
  sys.path.extend(TEST_LIBRARY_PATHS)

```

The webap2-1.2.3 module path is listed explicitly, together with a bunch of other modules. Giving a quick look inside the lib/ folder where modules are present, I noticed that for some modules there are multiple version sitting side by side. It seems that python doesn't just load the latest version by default.

I see to different approaches to address this problem:

1. Replicate the google_appengine/run_tests.py configuration and manipulate sys.path
2. Manipulate PYTHONPATH environment variable listing specific module versions for the ambiguous one

Both approaches have a problem: I could forget to reference the next version when it comes out.

I implemented the first and noticed a very strange behavior: on a particular test where I use mock.patch for a function that I expect to be called by the functionality being tested, the expect_to_be_called_with() stopped working, printing the .mock_calls list gives an empty list. This is something that needs further investigation.

I implemented the second and had a problem similar to the previous but now is the assertion on the patched method right before the previous that fails! There should be some difference in my local configuration and this is bad.
It is time to start using virtualenv...

# Appengine suggested test framework

I decided to give a try at the [appengine suggested approach](https://github.com/GoogleCloudPlatform/python-docs-samples/blob/master/appengine/standard/localtesting/runner.py) to see if the behavior in my machine is consistent (although it sound stupid because you know you should switch to virtualenv and put away alchemy).


# Virtualenv

virtualenv will keep consistent the set of libraries that are visible in your virtual environment, here is a quick list of commands you need to know.

```bash
# create a virtualenv named env
virtualenv env
# activate the environment just created
source env/bin/activate
# see libraries installed
pip list
# generate a dependency file
pip freeze --local > requirements.txt
# deactivate the environment
deactivate
```

# No luck

I still get a weird behavior for that particular test. Next step: isolate the weird behaviour I'm fighting and understand what goes wrong.


# The build script

This is some commands I played with

```bash
# seems unneeded when using virtualenv
pip install --upgrade setuptools
# alternatively could have used google-cloud-sdk
python scripts/fetch_gae_sdk.py
pip install -r requirements.txt --use-mirrors
# one approach to set environment variables
export PYTHONPATH=$PYTHONPATH:$(pwd)/google_appengine
# diagnose what python sees in his path
python -c "import sys; print(sys.path)"
```