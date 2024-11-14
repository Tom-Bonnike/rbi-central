# RBI Central

![Build Status](https://github.com/Shopify/rbi-central/workflows/CI/badge.svg)

## Overview

RBI Central is a cumulative repository of [RBI](https://sorbet.org/docs/rbi) ("Ruby Interface") file annotations for public gems which can be [imported by Tapioca](https://github.com/Shopify/tapioca#pulling-rbi-annotations-from-remote-sources). This repository can be used to annotate gem RBIs that Tapioca can’t represent fully and even fix type checking errors as a result.

## Usage

The RBIs in this repository are used by [Tapioca](https://github.com/Shopify/tapioca).

The [rbi-central](https://rubygems.org/gems/rbi-central) gem exists only to prevent name squatting - it should **not** be installed.

### Annotations

Annotations are hand-written RBI files for commonly used gems that tell Sorbet information about the gem (e.g., constants, ancestors, methods) that isn't already generated by Tapioca. In other words, annotations are shareable [shims](https://github.com/Shopify/tapioca#manually-writing-rbi-definitions-shims).

In this repository, annotations are placed in the `rbi/annotations` folder.

### Index

Each annotation from the `rbi/annotations` folder must be defined in the `index.json` file at the root of this repository:

```jsonc
{
    "gemA": { // gem name
        // optional: list of gems that need to be installed to test gemA RBI
        "dependencies": [
            "gemB",
            "gemC"
        ],
        // optional: list of files that need to be required to test gemA RBI
        "requires": [
            "file1",
            "file2",
        ]
    }
}
```

If you're copying this into your own `index.json`, make sure you strip out the comments.

See the index [validation schema](schema.json) for more details.

### Writing Annotations

#### Missing methods and shims

It is possible to allow necessary shims for non-existing runtime definitions by using comment tags:

* `@missing_method` to indicate that a method is delegated to another receiver using `method_missing`
* `@shim` to indicate that a definition doesn't actually exist at runtime but is needed to allow type checking

#### Versioned annotations

Many gems simultaneously maintain more than one version, often with different external APIs. If the gem's RBI
annotation file does not match the version being used in your project, it can result in misleading type checking
errors that slow down development.

Starting in version 0.1.10, the rbi gem supports adding gem version comments to RBI annotations, allowing us to
specify the gem versions that include specific methods and constants. [Tapioca](https://github.com/Shopify/tapioca)
version 0.15.0 and later will strip out any parts of the annotation file that are not relevant to the current gem
version when pulling annotations into a project.

##### Syntax

To use this feature, add a comment in the following format directly above a method or constant:

```ruby
# @version > 0.2.0
sig { void }
def foo; end
```

The comment must start with a space, and then the string `@version`, followed by an [operator](#operators) and
a version number. Version numbers must be compatible with Ruby's
[`Gem::Version` specification](https://ruby-doc.org/current/stdlibs/rubygems/Gem/Version.html).

Any method or constant that does not have a version annotation will be considered part of all versions.

For more information about Ruby gem versions, see the [Ruby documentation](https://ruby-doc.org/3.3.4/stdlibs/rubygems/Gem/Version.html).

##### Combining Versions

Version comments can use both "AND" and "OR" logic to form more precise version specifications.

###### AND

To specify an intersection between multiple version ranges, use a comma-separated list of versions. For example:

```ruby
# @version >= 0.3.4, < 0.4.0
sig { void }
def foo; end
```

The example above specifies a version range that starts at version 0.3.4 and includes every version up to 0.4.0.

###### OR

To specify a union between multiple version ranges, place multiple version comments in a row above the same method or
constant. For example:

```ruby
# @version < 1.4.0
# @version >= 4.0.0
sig { void }
def foo; end
```

The example above specifies a version range including any version less than 1.4.0 OR greater than or equal to 4.0.0.

### Pulling annotations

To pull relevant gem annotations into your project, run Tapioca's [`annotations` command](https://github.com/Shopify/tapioca#pulling-rbi-annotations-from-remote-sources) inside your project:

```shell
$ bin/tapioca annotations
```

## CI checks

The CI for this repository is written using GitHub Actions. Here is a list of currently available CI checks:

| CLI Command                       | Description                                                             |
| ----------------------------------| ----------------------------------------------------------------------- |
| `bundle exec repo check index`    | Lint `index.json` and check entries match `rbi/annotations/*.rbi` files |
| `bundle exec repo check rubygems` | Check new RBI files belong to public gems                               |
| `bundle exec repo check rubocop`  | Lint RBI files                                                          |
| `bundle exec repo check runtime`  | Check RBI annotations against runtime behavior                          |
| `bundle exec repo check static`   | Check RBI annotations against Tapioca generated RBIs and Sorbet         |

To avoid pushing a PR with errors, configure the `pre-push` git hook by running:

```shell
$ git config core.hooksPath gem/hooks
```

This git hook will run most of the CI checks and block the push if you don't pass all the tests.

### Runtime checks

Ensure the constants (constants, classes, modules) and properties (methods, attribute accessors) exist at runtime.

For each gem the test works as follows:

1. Create a temporary directory
2. Add a Gemfile pointing to the gem (and possible dependencies, listed in index key `dependencies`)
3. Add a Ruby script that:
   1. Requires the gem (and possible other requirements, listed in index key `requires`)
   2. Tries to `const_get` each constant defined in the RBI file
   3. Tries to call `instance_method` for each method and attribute accessor (or `method` for singleton methods) in the RBI file

### Static checks

Ensure the constants (constants, classes, modules) and properties (methods, attribute accessors) exist are not duplicated from Tapioca generated RBIs and do not create type errors when running Sorbet.

For each gem the test works as follows:

1. Create a temporary directory
2. Add a Gemfile pointing to the gem (and possible dependencies, listed in index key `dependencies`)
3. Generate the RBIs for gems using `tapioca gems`
4. Check duplication with RBIs from gems using `tapioca check-shims`
5. Check for type errors using `srb tc`

## Contributing

If you working on the gem using VS Code, change into the `gem/` directory then run `code .`.
This ensures that Sorbet and vscode-rdbg will work correctly.

Bug reports and pull requests are welcome on GitHub at https://github.com/Shopify/rbi-central.

This project is intended to be a safe, welcoming space for collaboration, and contributors are expected to adhere to the [Contributor Covenant](https://github.com/Shopify/rbi-central/blob/main/CODE_OF_CONDUCT.md) code of conduct.

## License

The gem is available as open source under the terms of the
[MIT License](https://github.com/Shopify/rbi-central/blob/main/LICENSE.txt).
