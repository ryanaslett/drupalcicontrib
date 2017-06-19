# Codebase Architectural Overview:

The DrupalCI testrunner is the program responsible for actually running
the tests on the drupalci testbots. DrupalCI testrunner is a symfony
console command line application with essentially one command - run.
(with some more helpers planned).

It has the following general responsibilities:

-   Assembling the codebase to be tested
-   Establishing the environment the tests need to run in
-   Analyzing and validating the codebase for correctness
-   Executing any assessments (tests)
-   Formatting and preserving build artifacts for later review

The testrunner has two main external dependencies : the
drupalci_environments repository, which is where all of the dockerfiles
for establishing environments are managed, and the drupal.org
infrastructure repository, where the Host AMI and Vagrant box VM are
built using packer. (Note, soon this will just be in the
drupalci_environments repo: https://www.drupal.org/node/2858975)

The high level flow of execution for the testrunner is as follows:

-   parse a build.yml definition file which determines which plugins
    will run, as well as their configuration
-   Iterate over the defined hierarchy of configured BuildTask plugins,
    executing each in turn
-   Preserving any artifacts and cleaning up after the build.

# Detailed floorplan of DrupalCI’s dungeon:

## BuildTask Plugins

The primary unit of execution in drupalci is a BuildTask Plugin
(DrupalCIPluginBuildTask). Each plugin is responsible for a defined
unit of work, like patching the codebase, or checking changed files for
syntax errors, or executing a particular testrunner. BuildTasks can be
divided into three categories, Stages, Phases, and Steps.

-   Stages are the highest level and represent the main
    responsibilities - codebase, environment, and assessment.
-   Phases are the next level down and act as groupings for the Steps,
    the lowest level. This hierarchy allows us to aggregate results into
    higher groupings and determine success if all child steps succeeded.
-   Steps are the individual build elements such as a particular type of
    assessment like simpletest, or a particular codebase validation step
    like eslinting.
-   It’s also somewhat easier to reason about the purpose of things,
    even though there are many blank classes that do very little (all of
    the phases). The key thing to see is that all BuildStages,
    BuildPhases, and BuildSteps are all considered to be BuildTasks, and
    all of them Inherit from BuildTaskBase.

## Build definition file (build.yml)

The build.yml file (/build_definitions) is the definition file that
contains the hierarchical list of plugin names and plugin configuration
options. Each key in the build.yml file either represents a BuildTask
plugin name, or a configuration value for a BuildTask plugin. The root
level is always the ‘build’, the second level are the BuildStage
plugins, the third level are the BuildPhase plugins, and the fourth
level are the BuildStep plugins.

## "Build" Objects

Plugins will need to interact with “Build Objects” - these are objects
that are non-specific to any plugin, but rather components typically
needed by all plugins, like the codebase, the ‘build’ itself, and the
‘environment’, and artifacts objects too.

## Providers

DrupalCI relies on dependency injection and the pimple container to
inject services into the build plugins. Yes, it does look an awfully lot
like a service locator pattern, but hey, its really not that shabby.
There are ServiceProvider definitions that define various services that
BuildTask plugins or Build Objects are going to need. Things like a
Docker service for communicating with the docker api, or a yaml parser
service for munging yaml. There are also services for the build objects
that we might also want to use, like the build, codebase, and database.

## Console Command

DrupalCI *is* a symfony console app, but it really only has one
command, currently - “run”. Eventually it will have some additional
local testing/configuration helpers but for now this is all it supports.

## Composer Plugin

There is a small composer plugin that sets everything up for
Codesniffing in development.

DrupalCI Application Run lifecycle:

The flow through drupalci begins with the symfony console command ‘run’
(DrupalCIConsoleCommandRun) - which does two things - generates
the build, and executes the build.

## Generating the build

Generating the build is a matter of parsing the build.yml file and
instantiating all the buildTask plugins such that they are all ready to
execute.


DrupalCI can gather a build.yml file in one of three ways:

`./drupalci run simpletest` -> if the argument after ‘run’ does not end
in .yml, drupalci will look in the /build_definitions folder for a file
of the same name, in this case it will use the ‘simpletest.yml’ build
definition.

`./drupalci run ./build.yml` -> if the argument after ‘run’ is a path
to a .yml file, drupalci will use that file directly as its build
definition.

`./drupalci run
https://dispatcher.drupalci.org/job/default/101/artifact/jenkins-default-101/artifacts/build.jenkins-default-101.yml`
-> if the argument after ‘run’ is a url to a .yml file, for example
on the dispatcher, then drupalci will retrieve that file and use it for
the build definition.

## Build.yml details

Build.yml files will contain a hierarchy of plugin names and
configuration values for the plugin.

This is a sample build.yml (the default simpletest yaml file). In the
following example shows that most of these plugins are using their
defaults and expecting to be overridden by environment variables, with
the exception of the concurrency value for simpletest.standard and
simpletest.js.

Additionally there is the concept of a ‘plugin label’ - because
sometimes you want to execute the same plugin twice with different
configuration, in this example “standard” and “js” on the simpletest
plugin are labels to distinguish between the simpletest execution and
the javascript functional tests (which also require 1 concurrency).
```yml
build:

  codebase:

   assemble_codebase:

     replicate:

     checkout_core:

     composer.core_install:

     composer_contrib:

     fetch:

     patch:

     update_dependencies:

 environment:

   startcontainers:

     runcontainers:

     start_phantomjs:

   create_db:

     dbcreate:

   assessment:

     validate_codebase:

       phplint:

       container_composer:

       phpcs:

     testing:

       simpletest.standard:

         concurrency: 31

       simpletest.js:

         concurrency: 1

         types: 'PHPUnit-FunctionalJavascript'
```

## Preparing the buildtasks:

A BuildTask plugin goes through the following steps to become “ready” to
execute:

`construct()->getDefaultConfiguration()->configure()->override_config()->inject()`

Plugins in drupalCI can have any number of configuration options
available, but every plugin should be able to execute by default. This
default configuration is set by ‘getDefaultConfiguration()’ on every
plugin. This also acts as a manifest for what is configurable. The
second step, ‘configure()’ is going to look for environmental overrides
to those default values (Set by various DCI_NAMESPACED environment
variables) someday this will also include command line switches as
another source of configuration. The third step is to ‘override_config’
-> *any* values that were specified in the build.yml file can be
considered to be “hard coded” and will always take precedence over
environment variables and defaults. The final step is to inject any
remaining service dependencies into the plugin so that it has whatever
objects it needs to execute.

## Executing the build

The build recurses through the hierarchy of plugins, and each plugin
follows a flow of execution like so:

`start->setup->run->execute_children->complete->teardown->finish->return
to parent`

The start/setup and teardown/finish steps are where all universal plugin
activities happen (setting up directories, starting timers, outputting
start/stop to console etc).

The ‘run’ step of a plugin is where the bulk of all the work is going to
happen for a plugin. This is where the tests are run, patches are
applied etc.

If a plugin is a BuildStage or BuildPhase, it executes its children
after it runs.

Once a plugin’s children have all executed, a plugin then executes its
‘complete’ step. This is mostly useful for parent plugins but can
sometimes be handy for BuildStep plugins too.

## Filesystem and Environments

DrupalCI is designed to run most of its tasks either on the ‘Host’ -
things like building the codebase, or validating the code, or it runs
inside of docker containers with specific versions of php and databases
available - this is the Container environment, which is intended to
exercise the code in all the environmental combinations that Drupal
supports.

This adds a certain level of complexity that can be hard to reason about
because you end up with a lot of files and folders both inside of the
containers and outside of the containers, and you also have certain
folders bind mounted such that they are shared between the container and
the host.

Most of these folders have convenient semantic abstractions layered on
top of them, so ideally one does not spend much time thinking about what
the actual file/folder path is, or what the user/group permissions needs
to be depending on whether you are in container context or in host
context.

## Host Filesystem:

The host filesystem on the AMI as well as inside the vagrant box has the
following key directories:
```
├/opt/drupalci

 ├── composer-cache

 ├── drupal-checkout

 └── testrunner
```
/opt/drupalci contains a composer-cache, a copy of drupal core and a
copy of the testrunner code from when the box was last rebuilt.
```
/var/lib/drupalci/

├── coredumps

├── docker-tmp

├── drupal-checkout

└── workspace

 ├── remote_6d2e9dddf7d4dfefeb1f6252d0d86b59

 │ ├── ancillary

 │ ├── artifacts

 │ ├── database

 │ └── source

 └── local_a89cb40d80eb97927a0b8927fc8db71b
```
`/var/lib/drupalci` is a tmpfs mounted filesystem inside the box/on the
AMI. It is 70% of allocated memory.

`/var/lib/drupalci/coredumps` is where anything that segfaults inside the
docker containers will end up.

`/var/lib/drupalci/drupal-checkout` is a copy of
`/opt/drupalci/drupal-checkout` that gets put there *after* the tmpfs is
created.

`/var/lib/drupalci/workspace` is where any of the testruns are going to
end up.

The folders under that dir will all start with
remote_/local_/buildname_ depending on whether or not its a url/local
file/build_definition default file. It may also end up as the Jenkins
Build Tag.

Underneath each individual testrun’s directory are the following four
subfolders:

-   /ancillary - Ancillary work directory
-   /artifacts - Artifact Directory
-   /source - Source code Directory
-   /database - Database File Directory

## Containers and Environments - Inside vs outside.

There are currently/typically two containers per build: a php
“executable” container, and a “database” container. In addition to the
four workspace folders, the composer cache folder and the core dumps
folder is mounted to the containers.

To access the filesystem in the container (for saving artifacts, using
the work dir) you go through the Environment build object. Most of the
the local filesystem folders on the Host are located on the main “Build”
object. This is subject to change: see
[*https://www.drupal.org/node/2852371*](https://www.drupal.org/node/2852371)

# Developing new features for DrupalCI

All new plugins for extending DrupalCI first need to find the right
location in the hierarchy that is somewhat descriptive of what its
purpose is.

Once a home for the plugin has been located, you need to make sure it
can be found.

Give it a unique name. Currently there’s no easy way to ensure that
you’re not colliding with another plugin other than to check some build
files or look at all the plugins. Your plugin class should extend
BuildTaskBase, and implement BuildTaskInterface.

```php
/**

 * @PluginID("my_plugin")

 */

class MyPluginBuildStep extends BuildTaskBase implements
BuildTaskInterface {

}
```
Now your plugin can be added to build.yml files and it will work. But it
won't do anything—yet.

Implement the run() method of the BuildTaskInterface as the first thing.

```php
/**

 * @PluginID("my_plugin")

 */

class MyPluginBuildStep extends BuildTaskBase implements
BuildTaskInterface {

// Implement the inject method, and *call the parent*

// so that your buildstep plugin gets all of the dependencies

// It needs. Parent on buildtask plugins get the build, the io output
object, and the container itself by default.

 public function inject(Container $container) {

 parent::inject($container);

 // Here we are pretending that our Buildstep needs a codebase, like
maybe it does something with it.

 $this->codebase = $container['codebase'];

 }

 public function run(){

 // $this->io is a dependency injected into the BuildTaskBase.

 // It’s what lets you echo stuff

 $this->io->writeln(">info>Doing a bunch of things
>options=bold>Really loudly>/>>/info>”);

}
```

## Executing Commands

Most plugins will want to run commands, both on the host and inside the
testing environment.

If you wish to execute commands locally on the host, there are two
types: required commands that, if they fail, should abort the build, and
optional commands that if they fail, the build should continue
processing.

`// Execute a required command on the host. Failure aborts the build:`

`$this->execRequiredCommand($cmd, 'Composer config failure');`

execRequiredCommand takes in the command, and a short message to display
in the build outcome if the required command fails.

Example of a non-required command -always use the $this->exec() form
so that during testing you can skip actual execution of the exec, and
see that it was attempted.

`$cmd = "cd '$directory' && git log --oneline -n 1 --decorate";`

`$this->exec($cmd, $cmdoutput, $result);`

Sometimes the environment that the command is executed within needs to
match the testing environment, and the commands need to run inside of
the docker containers. Currently the ‘environment’ build object provides
access to those containers in order to execute commands:

`// Execute a command inside the php docker container`

`$result = $this->environment->executeCommands($commands);`

`// Execute a command inside a particular docker container`

`$result = $this->environment->executeCommands($commands,
$container[‘id’]);`

Both of those container commands return a CommandResult object which
contain the output, error, and return signal of those commands.

## Terminating the Build

When a Required command is executed on the host, it will automatically
terminate the build if it fails. However, if a command is executed on a
container, you will need to inspect the CommandResult object to
determine if it was successful, and if it was not, and the build should
not proceed, there is a method on the BuildTaskBase to terminate the
build. This method takes in two arguments, a short message that ends up
in the build outcome, and an extended message for the details of the
failure: For example:

`if ($patch->apply() !== 0) {
   $this->terminateBuild("Patch Failed to Apply", implode("n", $patch->getPatchApplyResults()));
 }`

## Preserving Artifacts

During the execution of a plugin, oftentimes there may be data that
would be valuable to inspect post build. Sometimes that’s the output of
a command, or a report file, and sometimes its intermediary work files
that are generated as part of the process of executing the plugin. In
order to save these files we have three methods built into the
BuildTaskBase for preserving them that eliminates any namespacing
conflicts.

If you have a filename on the host that you wish to save, you supply the
full path to that file, as well as the filename you wish to save it as
(as sometimes things are contextually named by their directory, like
composer/vendor/installed.json, and it might be better to rename the
file to composer-installed.json, as an example).

`$this->saveHostArtifact($filepath, $savename);`

If you do not have a file, but have a string you wish to save, you can
use saveStringArtifact to create the file for you.

`$this->saveStringArtifact($filename, $contents);`

Finally, if the artifact exists only inside the docker container
filesystems, you’ll want to use saveContainerArtifact with the full path
to have it store the file in the artifacts directory.

`$this->saveContainerArtifact($filepath, $savename);`

All artifacts will then end up in the workspace/artifacts directory,
each namespaced by the plugin that executed them.

## Plugin Configuration

Plugins can be controlled with configuration values. Configuration
values come from three sources: Defaults set on a plugin, Namespaced
DCI_ENVIRONMENT variables, and configuration values set in the
build.yml. A fourth source is planned (command line switches).

As a general rule, the configuration philosophy of DrupalCI is to avoid
configuritits wherever possible. We really want to keep configuration
down to a minimum, and only add options as use cases arise. If a value
can somehow be determined from the codebase or build environments, then
that value is not a good candidate for configuration.

But, plugins need options, so a plugin needs to define a two things:

### `getDefaultConfiguration();`

This method is intended to define the configuration keys that your
plugin cares about. It really only keeps track of the top level keys, so
it doesn’t concern itself with array contents or nested data structures.
Any configurable item in your plugin *should* be defined in
getDefaultConfiguration, even if it is blank, as this is what is used to
export the build.yml file artifact on test generation.

```php
/**
 * @inheritDoc
 */

public function getDefaultConfiguration() {

  return [
    'testgroups' => '--all',
    'concurrency' => 4,
    'types' => 'Simpletest,PHPUnit-Unit,PHPUnit-Kernel,PHPUnit-Functional',
    'url' => 'http://localhost/checkout',
    'color' => TRUE,
    'die-on-fail' => FALSE,
    'keep-results' => TRUE,
    'keep-results-table' => FALSE,
    'verbose' => FALSE,
    // testing modules or themes?
    'extension_test' => FALSE,
  ];

}
```

### `configure();`

This method looks for defined environment variables and overrides the
configuration variables based on those environment variables. Sometimes
there isn't a 1:1 mapping of environment variables to config options, or
the environment variable contains a serialized format of data that needs
to be parsed into individual values. In the following example,
DCI_TestItem needs to be parsed to figure out what it is trying to
accomplish.

```php
 /**
  * @inheritDoc
  */

 public function configure() {

   // Override any Environment Variables

   if (FALSE !== getenv('DCI_Concurrency')) {
     $this->configuration['concurrency'] = getenv('DCI_Concurrency');
   }

   if (FALSE !== getenv('DCI_RTTypes')) {
     $this->configuration['types'] = getenv('DCI_RTTypes');
   }

   if (FALSE !== getenv('DCI_RTUrl')) {
     $this->configuration['types'] = getenv('DCI_RTUrl');
   }

   if (FALSE !== getenv('DCI_RTColor')) {
     $this->configuration['color'] = getenv('DCI_RTColor');
   }

   if (FALSE !== getenv('DCI_TestItem')) {
     $this->configuration['testgroups'] = $this->parseTestItems(getenv('DCI_TestItem'));
   }

   if (FALSE !== getenv('DCI_RTDieOnFail')) {
     $this->configuration['die-on-fail'] = getenv('DCI_RTDieOnFail');
   }

   if (FALSE !== getenv('DCI_RTKeepResults')) {
     $this->configuration['keep-results'] = getenv('DCI_RTKeepResults');
   }

   if (FALSE !== getenv('DCI_RTKeepResultsTable')) {
     $this->configuration['keep-results-table'] = getenv('DCI_RTKeepResultsTable');
   }

   if (FALSE !== getenv('DCI_RTVerbose')) {
     $this->configuration['verbose'] = getenv('DCI_RTVerbose');
   }

 }
```
## Build Objects / Services

DrupalCI utilizes several build objects that are not unique to plugins
that provide data and functionality across the system to the plugins.
Each ‘service’ exists in the dependency injection container and is
accessable via the container. Each service is created by a
serviceprovider object ./src/DrupalCI/Providers/* is where these
objects live, with DrupalCIServiceProvider being the main one where
others need to be registered.

The services that a plugin can interact with are currently as follows:

-   Build - provides information and path info about the current build
-   Codebase - provides information about the build codebase, such as
    which files have been modified etc.
-   Console.io - provides a way to output info and errors while the test
    runs.
-   Environment - gives plugins a way to interact with the environment
    that the system under test will run in.
-   Database - provides connectivity to the database for the system
    under test.
-   Guzzle - gives plugins a way to connect and communicate to external
    services via http
-   Yaml - allows plugins to store and retrieve yaml definitions.
-   Docker - gives plugins a way to start/stop docker containers.

Most of these objects are “build” objects and exist under
src/DrupalCI/Build

**How to create a service for a dependency in the container
(Providers)**

Artifact/Codebase/Environment/BuildOutcome

How to create a service for a dependency in the container (Providers)

## Special Environment Variables

Most environment variables in drupalci are meant to provide a
configuration value to a BuildStep plugin. However, there are four
additional environment variables that alter the behavior of drupalci.

-   **DCI_Debug**

    -   When drupalci runs, normally it will clean up all working files,
        > the codebase, and core files (for the simpletest plugin). If
        > the DCI_Debug flag is set, it will not attempt to clean up
        > those directories.

-   **DCI_WorkingDir**

    -   When drupalci starts, normally it will be putting the build file
        > into /var/lib/drupalci (php is configured to have
        > sys_get_temp_dir return /var/lib/drupalci in the vagrantbox
        > as well as the testbot ami). Setting DCI_WorkingDir allows a
        > user running locally to specify a different directory (perhaps
        > one with more space).

-   **DCI_JobType**

    -   When executing ./drupalci run >jobtype>, if this
        > environment variable is set, then >jobtype> is set via
        > the environment. The jobtype corresponds to the build.yml file
        > that is used as a job template in the /build_definitions
        > directory. This is mostly a function of Jenkins, but probably
        > may be deprecated in the future in favor of direct execution.

-   **BUILD_TAG**

    -   The build_id of the running build (unique to each invocation of
        > drupalci) will be set to the jenkins BUILD_TAG if it is set.

## Testing Plugins

There are two ways to write tests for plugins to prove that they work,
and do not have unexpected side effects on the rest of the system

## Functional tests

Functional tests allow us to run a complete end to end test of drupalci
for a given scenario using all the required plugins to test that
scenario. In order to run that scenario we need to provide configuration
to the test. We can achieve that a few ways.

-   Using ENVs

    -   You can pass in environment variables to a test. This simulates
        execution of one of the default builds in build_definitions
        directory, with configuration provided by the environment
        variables. You can set environment variables in the start of
        the test by specifying $dciConfig as an array of
        DCI_>variable> ‘s
        ```php
          protected $dciConfig = [
           'DCI_UseLocalCodebase=/var/lib/drupalci/drupal-checkout',
           'DCI_LocalBranch=8.3.x',
           'DCI_LocalCommitHash=c187f1d',
           'DCI_JobType=simpletest',
           'DCI_TestItem=Url',
           'DCI_PHPVersion=php-7.0-apache:production',
           'DCI_DBType=mysql',
           'DCI_DBVersion=5.5',
           'DCI_CS_SkipCodesniff=TRUE',
           ];```

-   Using .yml files

    -   Rather than passing in ENV’s, we can start with a fully formed
         build.yml file, and include it as an argument to the test.
         This has advantages, and disadvantages. The advantage is you
         can take a failed drupalci test that needs problems fixed and
         use it directly as a test case. The disadvantage is that the
         build.yml acts as a defacto API, so it could add a potential
         maintenance burden if keys and configurations change over
         time. Another advantage of build.yml files is if you are
         testing a plugin that is earlier in the build, like codebase
         or environment plugins, you can trim off unnecessary parts of
         the build (like assesment/environmnent phases)

         To use this method, include the following directive in your
         tests:
         `$app_tester->run([ 'command' => 'run',
                            'definition' => 'tests/DrupalCI/Tests/Application/Fixtures/build.ContribD7ManyTestingDepsTest.yml',
         ], $options);`

         Where the definition file is relative to the root of the
         project.
-   Using .yml templates & ENV’s

    -   Sometimes a hybrid approach to testing works nicely, where you
         want a trimmed down or shortened version of the build.yml, but
         you want to run multiple tests. In this case you can add
         *both* environment variables and use a template to control
         the build test.
         See ./tests/DrupalCI/Tests/Application/PhpLint/* for examples
         of reusing a template while feeding different env values.

## Unit Tests

Faster tests can be written that exercise the logic *internal* to a
plugin. This is where a Unit test is nice. Please look at
./tests/DrupalCI/Tests/Plugin/* for examples of exercising an
individual plugin test. And if you have tips or tricks, submit a patch
to these docs to help improve them.
