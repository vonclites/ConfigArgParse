### Overview
Applications with more than a handful of user-settable options are
best configured through a combination of command line args, config files,
hard-coded defaults, and in some cases, environment variables.

Python's command line parsing modules like argparse have very limited support
for config files and environment variables, so this module extends argparse to
add these features.

### Features
- command-line, config file, env var, and default settings can now be defined,
  documented, and parsed in one go using a single API  (the order of precedence
  is: command line args > environment variables > config file values > defaults)
- config files can have .ini or .yaml style syntax (eg. key=value or key: value)
- user can specify a config file path using regular command line syntax
  (eg. -c config.txt) rather than the argparse-style @config.txt
- all argparse functionality is fully supported, so this module can serve as a
  drop-in replacement for argparse
- env vars and config file keys & syntax are automatically documented in the
  help message
- print_values() can be used to log values and their sources (eg. command line,
  env var, config file, or default) for improved reproducibility.
- lite-weight (simple API, no dependencies on 3rd-party libraries),  
- extensible (the following methods can be over-ridden to change config file
  and environment variable parsing: parse_config_file, get_possible_config_keys,
  convert_setting_to_command_line_arg)
- unittested using the tests that came with argparse, and using tox to test python versions >= 2.7


### Example

*my_script.py*:

```py
import configargparse

p = configargparse.ArgParser(
    default_config_files=['/etc/settings.ini', '/home/jeff/.my_settings'])
p.add('--genome', required=True, help='path to genome file')  # this option can be set in a config file because it starts with '--'
p.add('-v', help='verbose', action='store_true')
p.add('-c', '--my-config', required=True, help='config file path',
	is_config_file=True)
p.add('-d', '--dbsnp', help='known variants .vcf', env_var='DBSNP_PATH')  # this option can be set in a config file because it starts with '--'
p.add('vcf', nargs='+', help='variant file(s)')

options = p.parse_args()

print(p.format_help())
print("-----")
print(options)
print("-----")
p.print_values()    # useful for logging where different settings came from
```


*config.txt:*

```py
# settings for my_script.py
genome = HCMV     # cytomegalovirus genome
dbsnp = /data/dbsnp/variants.vcf
```

*Command line:*

```py
python my_script.py --genome hg19 --my-config config.txt  f1.vcf  f2.vcf
```

*Output:*
```py
usage: my_script.py [-h] --genome GENOME [-v] -c MY_CONFIG [-d DBSNP]
                    vcf [vcf ...]
Args that start with '--' (eg. --genome) can also be set in a config file
(/etc/settings.ini or /home/jeff/.my_settings or provided via -c) by using
.ini or .yaml-style syntax (eg. genome=value). Command-line values override
environment variables which override config file values which override
defaults.

positional arguments:
  vcf                   variant file
optional arguments:
  -h, --help            show this help message and exit
  --genome GENOME       path to genome file
  -v                    verbose
  -c MY_CONFIG, --my-config MY_CONFIG
                        config file path
  -d DBSNP, --dbsnp DBSNP
                        known variants .vcf [env var: DBSNP_PATH]
-----
Namespace(dbsnp='/data/dbsnp/variants.vcf', genome='hg19', my_config='config.txt', vcf=['f1.vcf', 'f2.vcf'], verbose=False)
-----
Command Line Args:   --genome hg19 --my-config config.txt f1.vcf f2.vcf
Config File (config.txt):
  dbsnp:             /data/dbsnp/variants.vcf
```

### Special Values

Normally, configargparse handles environment variables and config file values
by converting them to their corresponding command line arg. For example,
"key = value" results in "--key value"  being appended to the command line.

The following values (whether in a config file or an environment variable) are
handled in a special way:

* key = true - results in "--key" being appended to the command line. Key must be
    a boolean flag (eg. action="store_true" or similar).

* key = [value1, value2, ...] - results in "--key value1 --key value2" etc. being
	appended to the command line. Key must be defined as a list (eg. action="append").



### Global ArgumentParsers

To make it easier to configure applications that have multiple independent
modules, configargparse provides globally-available ArgumentParser instances via
configargparse.getArgumentParser('name') (similar to logging.getLogger('name')).

*main.py*
```py
import configargparse
import utils

p = configargparse.getArgumentParser()
p.add_argument("-x", help="Main module setting")
p.add_argument("--m-setting", help="Main module setting")
options = p.parse_known_args()   # using p.parse_args() here may raise errors.
```

*utils.py*
```py
import configargparse
p = configargparse.getArgumentParser()
p.add_argument("--utils-setting", help="Config-file-settable option for utils")
options = p.parse_known_args()
```


### Config File Syntax

Any command line arg that has a long version (eg. one that starts with '--')
can be set in a config file. For example, the arg "--color" can be set by
putting "color=green" in a config file. Here is the full range of valid syntax:

```yaml
    # this is a comment
    ; this is also a comment (.ini style)
    ---            # lines that start with --- are ignored (yaml style)
    -------------------
    [section]      # .ini-style section names are treated as comments

    # how to specify a key-value pair (all of these are equivalent):
    name value     # key is case sensitive: "Name" isn't "name"
    name = value   # (.ini style)  (white space is ignored, so name = value same as name=value)
    name: value    # (yaml style)
    --name value   # (argparse style)

    # how to set a flag arg (eg. arg which has action="store_true")
    --name
    name
    name = True    # "True" and "true" are the same

    # how to specify a list arg (eg. arg which has action="append")
	fruit = [apple, orange, lemon]
	indexes = [1, 12, 35 , 40]
```


### Aliases

The configargparse.ArgumentParser API inherits its class and method names from
argparse and also provides the following shorter names for convenience:

* configargparse.getArgParser()
* configargparse.getParser()
* p = configargparse.ArgParser()
* p = configargparse.Parser()
* p.add_arg(..)
* p.add(..)
* options = p.parse(..)


### Design Notes

Unit tests:

tests/test_configargparse.py contains custom unittests for features specific to this module (such as config file and env-var support), as well as a hook to load and run argparse unittests (see the built-in test.test_argparse module) but on configargparse in place of argparse. This ensures that configargparse will work as a drop in replacement for argparse.

Are unittests still passing:    ![Travis CI Status for zorro3/ConfigArgParse](https://api.travis-ci.org/zorro3/ConfigArgParse.svg?branch=master)  [![Analytics](https://ga-beacon.appspot.com/UA-52264120-1/ConfigArgParse/ConfigArgParse)](https://github.com/igrigorik/ga-beacon)

Previously existing modules (PyPI search keywords: config argparse):

* argparse (built-in module python v2.7+ )
    * Good:
        - fully featured command line parsing
        - can read args from files using an easy to understand mechanism
    * Bad:
        - syntax for specifying config file path is unusual (eg. @file.txt)and not described in the user help message.
        - default config file syntax doesn't support comments and is unintuitive (eg. --name\n value)
        - no support for environment variables
* ConfArgParse v1.0.15 (https://pypi.python.org/pypi/ConfArgParse/1.0.15)
    * Good:
        - extends argparse with support for config files parsed by ConfigParser
        - clear documentation in README
    * Bad:
        - config file values are processed using ArgumentParser.set_defaults(..) which means "required" and "choices" are not handled as expected. For example, if you specify a required value in a config file, you still have to specify it again on the command line.
        - doesn't work with python 3 yet
        - no unit tests, code not well documented
* appsettings v0.5 (https://pypi.python.org/pypi/appsettings)
    * Good:
        - supports config file (yaml format) and env_var parsing
        - supports config-file-only setting for specifying lists and dicts
    * Bad:
        - passes in config file and env settings via parse_args namespace param
        - tests not finished and don't work with python3 (import StringIO)
* argparse_config v0.5.1 (https://pypi.python.org/pypi/argparse_config/0.5.1)
    * Good:
        - similar features to ConfArgParse v1.0.15
    * Bad:
        - doesn't work with python3 (error during pip install)
* yconf v0.3.2 - (https://pypi.python.org/pypi/yconf/0.3.2)  - features and interface not that great
* hieropt v0.3 - (https://pypi.python.org/pypi/hieropt) - doesn't appear to be maintained, couldn't find documentation


Design choices:


1. all options must be settable via command line. Having options that can only
be set using config files or env. vars adds complexity to the API, and is not
a useful enough feature since the developer can split up options into sections
and call a section "config file keys", with command line args that are just
"--" plus the config key.
2. config file and env. var settings should be processed by appending them
to the command line (another benefit of #1). This is an easy-to-implement
solution and implicitly takes care of checking that all "required" args are
provied, etc., plus the behavior should be easy for users to understand.
3. don't override convert_arg_line_to_args so that configargparse can pass all
argparse unit tests.
4. in terms of what to allow for config file keys, the "dest" value of an option
can't serve as a valid config key because many options can have the same dest.
Instead, since multiple options can't use the same long arg  (eg.
"--long-arg-x"), let the config key be either "--long-arg-x" or "long-arg-x".
This means the developer can allow only a subset of the command-line args to
be specified via config file (eg. short args like -x would be excluded).
Also, that way config keys are automatically documented whenever the command
line args are documented in the help message.
5. don't force users to put config file settings in the right .ini [sections].
This doesn't have a clear benefit since all options are command-line settable,
and so have a globally unique key anyway. Enforcing sections just makes things
harder for the user and adds complexity to the implementation.
6. if necessary, config-file-only args can be added later by implementing a
separate add method and using the namespace arg as in appsettings_v0.5

Relevant sites:

- http://stackoverflow.com/questions/6133517/parse-config-file-environment-and-command-line-arguments-to-get-a-single-coll
- http://tricksntweaks.blogspot.com/2013_05_01_archive.html
- http://www.youtube.com/watch?v=vvCwqHgZJc8#t=35

