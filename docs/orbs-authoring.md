# Authoring and publishing orbs
This document largely covers the tooling and flow of authoring and publishing your own orbs to the orb registry. Check out the [documentation on using orbs](using-orbs.md) for more details on how to pull orbs into your build configuration.

Orbs can be authored [inline](inline-orbs.md) in your `config.yml` file or authored separately and then published to to the orb registry for reuse across projects. The main focus of this document is how to _publish_ orbs for use across projects. For developing inline orbs, check out the [Writing Inline Orbs](inline-orbs.md) document.

### [WARNING] Orbs are always world-readable
All published orbs can be read and used by anyone. They are _not_ limited to just the members of your organization.
In general, we strongly recommend that you do not put secrets or other sensitive variables into your configuration. Instead, use contexts or project environment variables and reference the names of those environment variables in your orbs.

## Prerequisites
To start orb publishing, you will need to opt-in to the new 3rd Party Software terms and turn on orb publishing for your organization. Only an organization admin can do this from the [organization Settings page](https://circleci.com/docs/2.0/settings/#organization-settings-page). On the Security tab you will find the form for opting in.

## Concepts in orb publishing

### Orb registry
Each installation of CircleCI has a single orb registry. The registry on circleci.com serves as the master source for all certified namespaces and orbs and is the only orb registry that users of circleci.com can use.

When orb features come to the server installation of CircleCI, each installation will have its own registry that operators can control. 

> NOTE: as of September 2018 the orb features are not yet supported on server installations (TODO: update this doc if orbs are now on server installs).

### Namespaces
Namespaces are used to organize a set of orbs. Each namespace has a unique and immutable name within the registry, and each orb in a namespace has a unique name. For instance, the orb `circleci/rails` means the `rails` orb in the `circleci` namespace, which can coexist in the registry with an orb called `somenamespace/rails` because they are in separate namespaces. 

Namespaces are owned by organizations. Only organization administrators can create namespaces.

Organizations are, by default, limited to claiming only one namespace. This policy is designed to limit name-squatting and namespace noise. If you require more than one namespace please contact your account team at CircleCI.

### Dev vs. production orbs
Versions of orbs can be added to the registry either as development versions or production versions. Production versions are always a semver like `1.5.3`, and development versions can be tagged with a string and are always prefixed with `dev:`.

#### Dev and production orbs have different security profiles:

* Only organization administrators can publish production orbs. 

* Any member of an organization can publish dev orbs in namespaces.

#### Dev and production orbs have different retention and mutability characteristics:

* Dev orbs are mutable and expire. Anyone can overwrite any development orb who is a member of the organization that owns the namespace in which that orb is published.

* Production orbs are immutable and long-lived. Once you publish a production orb at a given semver you may not change the content of that orb at that version. To change the content of a production orb you must publish a new version. We generally recommend using the `orb publish increment` and/or the `orb publish promote` commands in the `circleci` CLI when publishing orbs to production.

#### Dev and production orbs have different versioning semantics
Development orbs are tagged with the format `dev:<< your-string >>`. Production orbs are always published using the [semantic versioning](https://semver.org/) ("semver") scheme.

In development orbs, the string label given by the user has the following restrictions:

Up to 1023 of the following characters:
  - Numbers `0-9`
  - Lowercase letters `a-z`
  - Dashes `-`
  - Underscores `_`
  - Periods `.`

Examples of valid development orb tags:
- Valid:
  - "dev:mybranch"
  - "dev:2018_09_01"
  - "dev:1.2.3-rc1"
- Invalid
  - "dev: 1" (No spaces allowed)
  - "dev:A" (No capital letters)
  - "1.2.3-rc1" (No leading "dev:")

In production orbs you must use the form `X.Y.Z` where `X` is a "major" version, `Y` is a "minor" version, and `Z` is a "patch" version. For instance, 2.4.0 implies the major version 2, minor version 4, and the patch version of 0. 

While we do not enforce it strictly, we strongly recommend that when versioning your production orbs you use the standard semver convention for deciding what major, minor, and patch should mean semantically:

MAJOR: when you make incompatible API changes,
MINOR: when you add functionality in a backwards-compatible manner, and
PATCH: when you make backwards-compatible bug fixes.

### Using orbs in orbs and register-time resolution
You may use an `orbs` stanza inside an orb, providing a way to pull in other orbs' elements into your orb. The scoping rules are the same as orbs in configuration, and you declare inline orbs in an orb.

NOTE that because production orb releases are immutable the system will resolve all orb dependencies at the time you register your orb rather than the time you run your build (eager resolution). For example, imagine you go to publish orb `foo/bar` at version `1.2.3`, and that orb you are publishing has, internally, an `orbs` stanza that imports another orb referenced as `biz/baz@volatile`. Let's imagine also that the latest version at the time you want to do this is `2.1.0`. At the time you register `foo/bar@1.2.3` the system will resolve `biz/baz@volatile` as the latest version and include its elements directly into the packaged version of `foo/bar@1.2.3`. Now imagine that afterwards `biz/baz` receives a new release that is now `3.0.0`. Anyone using `foo/bar@1.2.3` will not see the change in `biz/baz@3.0.0` until `foo/bar` is published at a higher version. 

### Yanking production orbs
In general, we prefer to never delete production orbs that were published as Open because it harms the reliability of the orb registry as a source of configuration, and the trust of all orb users. We recognize there are circumstances in which a production orb ought be deactivated and/or expunged. 

If you need to yank an orb for emergency reasons please contact CircleCI (NOTE: IF YOU ARE YANKING BECAUSE OF A SECURITY CONCERN PLEASE PRACTICE RESPONSIBLE DISCLOSURE: https://circleci.com/security/).

### Using the CLI to author and publish orbs
The `circleci` CLI has several handy commands for managing your orb publishing pipeline. The simplest way to learn the CLI is to [install it](https://circleci.com/docs/2.0/local-cli/#installing-the-circleci-public-cli-from-scratch-on-macos-and-linux-distros) and run `circleci help`. Below are some of the most pertinent commands for publishing orbs:

```
circleci namespace create <name> <vcs-type> <org-name> [flags]

circleci orb create <namespace>/<orb> [flags]

circleci orb validate <path> [flags]

circleci orb publish <path> <namespace>/<orb>@<version> [flags]

circleci orb publish increment <path> <namespace>/<orb> <segment> [flags]

circleci orb publish promote <namespace>/<orb>@<version> <segment> [flags]
```

For full reference use the `help` command inside the CLI, or visit [https://circleci-public.github.io/circleci-cli/circleci_orb.html](https://circleci-public.github.io/circleci-cli/circleci_orb.html) (TODO: make this a link to the main docs once that info is there).

### Getting started template

You can use this template to get started with authoring orbs. It includes each of the three top-level concepts of orbs.  Any orb can be equally expressed as an inline orb definition. It will generally be simpler to iterate on an inline orb and use `circleci config process .circleci/config.yml` to check whether your orb usage matches your expectation.


```yaml
version: 2.1
orbs:
  inline_example:
    jobs:
      my_inline_job:
        parameters:
          greeting_name:
            type: string
            default: olleh
        executor: my_inline_executor
        steps:
          - my_inline_command:
              name: <<parameters.greeting_name>>
    commands:
      my_inline_command:
        parameters:
          name:
            type: string
        steps:
          - run: echo "hello <<parameters.name>>, from the inline command"
    executors:
      my_inline_executor:
        parameters:
          version:
            type: string
            default: "2.4"
        docker:
          - image: circleci/ruby:<<parameters.version>>

workflows:
  build-test-deploy:
    jobs:
      - inline_example/my_inline_job:
          name: build
      - inline_example/my_inline_job:
          name: build2
          greeting_name: world

```

