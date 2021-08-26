## What is this?

I'm a big fan of [Dhall][] in general, because:

 - It's an aggressively simple language, so there's not that much to learn
 - It's pure and functional (all it does is compute config, it's not for writing programs)
 - It's strongly typed, which hugely increases confidence compared to editing pages of YAML

In addition, Dhall turns out to be an _excellent_ way to manage CI boilerplate.

The problem with boilerplate is that you can't abstract it away. We typically end up copy/pasting or using generators, both of which miss out on later improvements made to the source. Ideally we could do what we normally do in languages, which is use libraries.

But when you're talking about github workflow files, travis configuration files, Dockerfiles, cloudbuild files and all the other CI configuration, it's impractical to do that. You'd have to pick a language, and make sure that's installed. Then probably a package manager. _Then_ you'd have to put together a bit of a framework for generating various files, and sort out package publication for anything you want to share between projects.

We need something that's simple, appropriate for the task and supports super lightweight dependency management. Dhall is exactly that:

 - Frictionless dependency management: you can import dhall code from the internet (safely, and with efficient caching)
 - Lightweight and simple to install (a couple static binaries totalling a dozen mb)
 - Purpose-built: dhall's stated goal is to replace YAML. Which is (mostly) what we're doing!

## :warning: WARNING: `dhall-render` overwrites the output directory

When run, it will replace whatever is currently present in the output directory (`generated` by default) with generated files. You should never set `dhall-render`'s output directory to something which already contains useful files, **they will be deleted**.

Pointing this to an existing directory (effectively deleting it) is an unfortunate footgun to enable, but unfortunately it's a byproduct of an otherwise effective system - it's simple and successfully garbage collects unwanted files regardless of how you got into your current state.

## How does it work?

The idea is that we can have one big dhall expression to define "all the generated files". This is super powerful - thanks to dhall's remote imports, you can use whatever types and abstractions you need by importing straight from the internet (it even supports private Github repositories).

The expression contains a map of `path -> file expression`, each of which is a dhall value (containing the file contents as well as some metadata).

We render this tree of files (into a `generated/` directory for cleanliness), then install symlinks in the workspace, pointing to the various generated files.

There's options to control the generation - you can mark files as executable, or use a plain file instead of a symlink (e.g. to workaround Github actions choking on symlinks).

And it's self-hosting. You can use the `SelfInstall` module to add this tool _as a generated file in your repo_, eliminating manual setup. You can even pre-bake the default file path to use, resulting in a custom script which doesn't need any arguments.

## How should I bootstrap it?

The quickest way is with this one-liner:

```
curl -sSL https://raw.githubusercontent.com/timbertson/dhall-render/master/bootstrap.sh | bash
```

That'll get you set up with a default `dhall/files.dhall` (a copy of [bootstrap/files.dhall](./bootstrap/files.dhall)), run the initial render to install `dhall-render` itself, and pin your `dhall-render` version to a specific commit.

If you'd rather set it up manually, make a copy of [bootstrap/files.dhall](./bootstrap/files.dhall) in `dhall/files.dhall`, then run:

```
echo '(./dhall/files.dhall).files.`dhall/render`.contents' | dhall text | ruby
```

You should then run `./dhall/fix --freeze ./dhall/files.dhall` to pin your `dhall-render` version.

## Do I check in the generated files?

Yes, typically. One of the main uses is for files which must be in the repository, as a contract (e.g. `.travis.yml` or `.github/workflows/*.yml`). So for those you have no choice.

If you mark files as "linguist-generated" (see [./gitattributes][]), they'll be hidden by default in Github pull request diffs, which can be convenient.

In order to make sure your generated files remain in sync with the source expressions, you can follow a process outlined in [the dhall manual](https://github.com/Gabriel439/dhall-manual/blob/e19a35fbfb509fa6447fa9c53e8bd96f9b83e584/manuscript/05-SynchronizeFiles.md).

## Got examples?

[Indeed I do](./examples/)

## What can I do with the SelfInstall module?

You can use the `exe` attribute to get the default `dhall-render` executable (ruby script).

If you don't use `files.dhall` as your file expression, you can use `SelfInstall.makeExe SelfInstall::{ path = "path/to/files.dhall" }` to create a `self-install` with a different default path.

There's also the `fix` attribute, which is the (not stable but handy) [./maintenance/fix][] script. This walks the current directory (or arguments) and by default evaluates and formats any `.dhall` files it finds. You can pass `--lint`, `--freeze` etc to perform other operations instead.

## How can I use a local copy of an import during development?

When working on dhall libraries, it's common to want to develop against a local copy while making changes.

There's no good workflow for this built into dhall, so `dhall-render` provides a utility script (`dhall/local`) for this purpose. You should use dhall > 1.40.0, as prior versions will silently fall back to the public version if there is a type error in your local version.

To conditionally import either the local or public version of a library, you can use this pattern:

```dhall
let Dependency =
        (\(local : Bool) -> ../local-directory/package.dhall ? local "import failed")
          env:DHALL_LOCAL
      ? https://example.org/package.dhall
```

And then run whatever you need as a subcommand of `dhall/local`:

```
./dhall/local ./dhall/render
```

This will run the command with `DHALL_LOCAL` set to `True`.

**Use in libraries**:

If you're using this pattern in a library, you may prefer to scope this code (so that someone can use `local` to import your code, without also needing your dependencies available locally):

```dhall
let Dependency =
        (\(local : Bool) -> ../local-directory/package.dhall ? local "import failed")
          env:DHALL_LOCAL_MYLIB
      ? https://example.org/package.dhall
```

The `DHALL_LOCAL_MYLIB` environment variable will only be set if the user opts in by passing the `mylib` scope to the `local` script, i.e.:

```
./dhall/local -s mylib ./dhall/render
```

Multiple scopes should be comma-separated in a single argument, e.g. `./dhall/local -s foo,bar,baz ./dhall/render`

**Error reporting**:

Note that if the local import fails to load (a missing file, type error, etc), you will unfortunately
not get the full error, you will only get:

```
  Error: Not a function

      local "import failed"
```

When this happens you'll need to manually try to evaluate the local version in isolation to see the real error, e.g. `dhall ../local-directory/package.dhall`.


**Use outside the terminal**:

If you need to integrate this into your editor or other environment, you can manually export `DHALL_LOCAL=True` (or `DHALL_LOCAL_<SCOPENAME>=True` for specific scopes).

## How do I get more details about type errors?

`dhall-render` simply runs `dhall-to-json` on your `files.dhall` expression, and processes the results. If you have type errors, you can run `dhall --file files.dhall --explain` to get more details.

## Requirements

`dhall-to-json` and `ruby`

## How does it make you feel?

Honestly, it's _liberating_. Until I started down this path, I didn't realise how much mental friction I had with CI boilerplate. I was always unmotivated to fix things or improve my workflow, because it was tedious and I knew I'd have to completely redo it for every repo where I wanted the benefit.

Now, I feel like it's actually worth making proper reusable abstractions for this stuff because I can actually leverage it across as many repos as I need, and only have to maintain it once.

## Is it for me?

If you only have a couple of repos, probably not. I have a great deal of personal and work repos that I deal with, and for me the thought that each one doesn't have to reinvent the wheel is very satisfying.

## Downsides

It's not _zero_-dependency, obviously -- you can't get around your contributors needing dhall in order to change things. Many work environments have ways to make this simple (e.g. shared Docker tooling). But for public repos it depends what environment / tooling you can assume your users have.

## How stable is this?

I wrote it quickly, and it's only got a handful of automated tests. But it's very simple, the examples work, and I use it daily.

## Why is it written in Ruby?

It was expedient, because I want to use it in a workplace where ruby is ubiquitous. Maybe one day I'll rewrite it in Haskell, but then I'd need to set up more infrastructure in order to actually automate and distribute binaries.

[dhall]: https://dhall-lang.org/

