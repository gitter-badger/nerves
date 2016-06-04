## Systems

Nerves System dependencies are a collection of configurations to be fed into the the system build platform. Currently, Nerves Systems are all built using the Buildroot platform. The project structure of a Nerves System is as follows:

```
# nerves_system_*
mix.exs
nerves_defconfig
nerves.exs
rootfs-additions
VERSION
```

The mix file will contain the dependencies the System has. Typically, all that is included here is the Toolchain and the build platform. Here is an example of the Raspberry Pi 3 `nerves_system` definition:

```
defmodule NervesSystemRpi3.Mixfile do
  use Mix.Project

  @version Path.join(__DIR__, "VERSION")
    |> File.read!
    |> String.strip

  def project do
    [app: :nerves_system_rpi3,
     version: @version,
     elixir: "~> 1.2",
     compilers: Mix.compilers ++ [:nerves_system],
     description: description,
     package: package,
     deps: deps]
  end

  def application do
   []
  end

  defp deps do
    [{:nerves_system, "~> 0.1.2"},
     {:nerves_system_br, "~> 0.5.0"},
     {:nerves_toolchain_arm_unknown_linux_gnueabihf, "~> 0.6.0"}]
  end

  defp package do
   [maintainers: ["Frank Hunleth", "Justin Schneck"],
    files: ["LICENSE", "mix.exs", "nerves_defconfig", "nerves.exs", "README.md", "VERSION", "rootfs-additions"],
    licenses: ["Apache 2.0"],
    links: %{"Github" => "https://github.com/nerves-project/nerves_system_rpi3"}]
  end

end

```

Nerves Systems have a few requirements in the mix file:
1. The compilers must include the `:nerves_system` compiler after the `Mix.compilers` have executed.
2. There must be a dependency for the toolchain and the build platform.
3. You need to list all files in the `package` `files:` list so they are present when downloading from Hex.

## Package Configuration

In addition to the mix file, Nerves packages read from a special `nerves.exs` configuration file in the root of the package names. This file contains configuration information that Nerves loads before any application or dependency code is compiled. It is used to store metadata about a package. Here is an example from the `nerves.exs` file for `rpi3`:

```
use Mix.Config

version =
  Path.join(__DIR__, "VERSION")
  |> File.read!
  |> String.strip

config :nerves_system_rpi3, :nerves_env,
  type: :system,
  mirrors: [
    "https://github.com/nerves-project/nerves_system_rpi3/releases/download/v#{version}/nerves_system_rpi3-v#{version}.tar.gz"],
  build_platform: Nerves.System.Platforms.BR,
  build_config: [
    defconfig: "nerves_defconfig",
    package_files: [
      "rootfs-additions"
    ]
  ]

```

There are a few important and required keys present in this file:

**type** The type of Nerves Package. Options are: `system`, `system_compiler`, `system_platform`, `system_package`, `toolchain`, `toolchain_compiler`, `toolchain_platform`.

**mirrors** The URL(s) of cached assets. For nerves systems, we upload the finalized assets to Github releases so others can download them.

**build_platform** The build platform to use for the system or toolchain.

**build_config** The collection of configuration files. This collection contains the following keys:

  * **defconfig** For `Nerves.System.Platforms.BR`, this is the BuildRoot defconfig fragment used to build the system.

  * **kconfig** BuildRoot requires a `Config.in` kconfig file to be present in the config directory. If this is omitted, a default empty file is used.

  * **package_files** Additional files required to be present for the defconfig. Directories listed here will be expanded and all subfiles and directories will be copied over, too.

## Building Nerves Systems

Nerves system dependencies are light-weight, configuration-based dependencies that, at compile time, request to either download from cache, or locally build the dependency. You can control which route `nerves_system` will take by setting some environment variables on your machine:

`NERVES_SYSTEM_CACHE` Options are `none`, `http`, `local`

`NERVES_SYSTEM_COMPILER` Options are `none`, `local`

Currently, Nerves systems can only be compiled using the `local` compiler on a specially-configured Linux machine. For more information on what is required to set up your host Linux machine, you can read the `nerves_system_br` [Install Page](https://github.com/nerves-project/nerves_system_br/blob/master/README.md)

Nerves cache and compiler adhere to the `Nerves.System.Provider` behaviour. Therefore, the system is laid out to allow additional compiler and cache providers, to facilitate other options in the future like Vagrant or Docker. This will be helpful when you want to start a BuildRoot build on your Mac or Windows host machine.

### Using Local Cache Provider

Nerves systems can take up a lot of space on your machine. This is because the dependency needs to be fetched for each project | target | env. To save space, you can enable the local cache.

```
$ export NERVES_SYSTEM_CACHE=local
```

With the local cache enabled, Nerves will attempt to find a cached version of the system in the cache dir. The default cache dir is located at `~/.nerves/cache/system` You can override this location by setting `NERVES_SYSTEM_CACHE_DIR` env variable.

If the system your project is attempting to use is not present in the cache, mix will prompt you asking if you would like to download it.

```
$ mix compile
...
==> nerves_system_rpi3
[nerves_system][compile]
[nerves_system][local] Checking Cache for nerves_system_rpi3-0.5.1
nerves_system_rpi3-0.5.1 was not found in your cache.
cache dir: /Users/jschneck/.nerves/cache/system

Would you like to download the system to your cache? [Yn] Y
```

This will invoke the http provider and attempt to resolve the dependency.

## Custom Nerves Systems

There may come a time when the pre-build Nerves Systems don't meet your needs. Maybe you want to include one or more packages or you want to run on hardware that isn't in the list of [Nerves-supported Targets](https://hexdocs.pm/nerves/targets.html) yet.
The easiest way to create a new Nerves System is to check out [`nerves_system_br`](https://github.com/nerves-project/nerves_system_br) and create a configuration that contains the packages and configuration you need.
Once you get this working and booting on your Target, you can copy the configurations and files back into a new mix project following the structure described above.

Nerves systems invoke a compiler in `nerves_system` which, through an [Elixir Behaviour](http://elixir-lang.org/getting-started/typespecs-and-behaviours.html) called `Nerves.System.Provider`, calls the provider specified by the `NERVES_SYSTEM_COMPILER` and `NERVES_SYSTEM_CACHE` environment variables.
The default behaviour is to pull these items from the HTTP cache, which uses the mirrors defined in the Nerves package configuration.

If you want to force it to build locally, you will need to be on a Linux machine and configure your environment with:

```
NERVES_SYSTEM_CACHE=none
NERVES_SYSTEM_COMPILER=local
```

There are currently a few issues when invoking Buildroot through the Mix compiler, so it may be easier to change to the `nerves_system_br` directory and run `create-build.sh` with a reference to the `defconfig` location of your new Nerves system derivative.

* the `defconfig` and assets in `nerves_system_*/` are copied to the output directory at `_build/env/nerves/system/config`.
  This will change in an upcoming release, since we are no longer going to pursue `defconfig` fragments (`system_ext`) but rather full package management.
  This means that if you were to navigate to the output directory and go through the `make menuconfig` ... `make savedefconfig` process, the changes would actually be saved to the copied configuration located in the `_build/env/nerves/system/config` directory instead of changing the original.

* Whenever the Mix compiler invokes Buildroot, it will forcefully clean the `output` directory and build from scratch.

### Supporting New Target Hardware

If you're trying to support a new Target, there's a bit more work involved, depending on how mature the support for that hardware is in the Buildroot community.

If you can find an existing Buildroot configuration for your intended hardware:

1.  Check out their full buildroot and procedure and get their stuff to boot.

2.  Clone buildroot from their main repo and checkout the tag nerves is currently at:
    https://github.com/nerves-project/nerves_system_br/blob/2d6f76406813734377bc9dad4dd5017e81ffa129/create-build.sh#L16

3. Figure out where they forked off

4. Compare the differences between the two to determine what standard things they changed.

  * Anything added like packages and board configs can be copied into `nerves_system_br`.
  * Look for patches to existing packages that are needed.

5. Branch buildroot and bring in the relevant changes.

6. Create a Git patch in your buildroot location and copy that patch to https://github.com/nerves-project/nerves_system_br/tree/2d6f76406813734377bc9dad4dd5017e81ffa129/patches (making sure to adhere to the numerical ordering).

> TODO: what is the Git command to do that?

7. Create a new defconfig which mimics theirs, and get `nerves_system_br` to build it.

> NOTE: You probably want to disable any userland packages that may be included by default to avoid distraction.

8. Wrap it all up into a new Nerves system.

Once you know that you “got it right” (i.e. it still boots and things look good), look at how the [Travis CI configuration](https://github.com/nerves-project/nerves_system_rpi/blob/master/.travis.yml) works.
This is how tagged Systems are built and make their way to the releases assets in Github, thus allowing others to use them as cached assets.

