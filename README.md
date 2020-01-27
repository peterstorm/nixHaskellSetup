# nixHaskellSetup
Start a new project

There are multiple starting options to create a new project. The most common one is certainly to use cabal-install. Another popular option is to use stack. stack adds a layer on top of cabal-install and uses fixed set of libraries known to compile together. Another method is to nix to handle the dependencies and use cabal-install for the rest. That final choice is often considered as the most complex and difficult for beginners. Still this is the one I find the most elegant. This is the method I will use in this article.

```
{ nixpkgs ? import (fetchTarball https://github.com/NixOS/nixpkgs/archive/19.09.tar.gz) {} }:
let
  inherit (nixpkgs) pkgs;
  inherit (pkgs) haskellPackages;

  haskellDeps = ps: with ps; [
    base
    protolude
    containers
  ];

  ghc = haskellPackages.ghcWithPackages haskellDeps;

  nixPackages = [
    ghc
    pkgs.gdb
    haskellPackages.cabal-install
  ];
in
pkgs.stdenv.mkDerivation {
  name = "env";
  buildInputs = nixPackages;
  shellHook = ''
     export PS1="\n\[[hs:\033[1;32m\]\W\[\033[0m\]]> "
  '';
}
```

Still, you shall not be intimidated. Look:

    To create a new project the steps will be:
        run nix-shell (to have cabal executable in your PATH)
        run cabal install -i and answer a few questions
        copy a few .nix files in your project directory
        run another nix-shell in your new directory this time to enter in the local dev env of your new project.
    To add a new library:
        Just add it in the .cabal file, and enter again in your nix-shell.

I will just walk you through all the steps in detail. And mostly I will tell you not to take care about most warning messages. For our end-goal, those are mostly noise. I am aware of the level of complexity that it looks like at first. But really most of the apparent complexity is due to poor naming convention and not to any fundenmental core difficulty.
Bootstrap a project template files

    put the shell.nix file in some directory
    start nix-shell --pure
    in the nix shell create a new directory and then
    cabal init -i
    You should use the default value for most questions except:
        Should I generate a simple project with sensible defaults? [default: y] n
        the package should build "Library AND Executable" (choice 3)
        Cabal specification 2.4 (choice 4)
        Application directory choose app (choice 3)
        Library directory choose lib (choice 3)
        Add informative comments, choose yes.

Here is a full interaction:

~/dev/hsenv> nix-shell

[hs:hsenv]> mkdir my-app

[hs:hsenv]> cd my-app/

[hs:my-app]> cabal init -i
Warning: The package list for 'hackage.haskell.org' does not exist. Run 'cabal
update' to download it.
Should I generate a simple project with sensible defaults? [default: y] n
What does the package build:
   1) Executable
   2) Library
   3) Library and Executable
Your choice? 3
What is the main module of the executable:
 * 1) Main.hs (does not yet exist, but will be created)
   2) Main.lhs (does not yet exist, but will be created)
   3) Other (specify)
Your choice? [default: Main.hs (does not yet exist, but will be created)]
Please choose version of the Cabal specification to use:
 * 1) 1.10   (legacy)
   2) 2.0    (+ support for Backpack, internal sub-libs, '^>=' operator)
   3) 2.2    (+ support for 'common', 'elif', redundant commas, SPDX)
   4) 2.4    (+ support for '**' globbing)
Your choice? [default: 1.10   (legacy)] 4
Package name? [default: my-app]
Package version? [default: 0.1.0.0]
Please choose a license:
   1) GPL-2.0-only
   2) GPL-3.0-only
   3) LGPL-2.1-only
   4) LGPL-3.0-only
   5) AGPL-3.0-only
   6) BSD-2-Clause
 * 7) BSD-3-Clause
   8) MIT
   9) ISC
  10) MPL-2.0
  11) Apache-2.0
  12) LicenseRef-PublicDomain
  13) NONE
  14) Other (specify)
Your choice? [default: BSD-3-Clause]
Author name? [default: Yann Esposito (Yogsototh)]
Maintainer email? [default: yann.esposito@gmail.com]
Project homepage URL?
Project synopsis?
Project category:
 * 1) (none)
   2) Codec
   3) Concurrency
   4) Control
   5) Data
   6) Database
   7) Development
   8) Distribution
   9) Game
  10) Graphics
  11) Language
  12) Math
  13) Network
  14) Sound
  15) System
  16) Testing
  17) Text
  18) Web
  19) Other (specify)
Your choice? [default: (none)]
Application (Main.hs) directory:
 * 1) (none)
   2) src-exe
   3) app
   4) Other (specify)
Your choice? [default: (none)] 3
Library source directory:
 * 1) (none)
   2) src
   3) lib
   4) src-lib
   5) Other (specify)
Your choice? [default: (none)] 2
Should I generate a test suite for the library? [default: y]
Test directory:
 * 1) test
   2) Other (specify)
Your choice? [default: test]
What base language is the package written in:
 * 1) Haskell2010
   2) Haskell98
   3) Other (specify)
Your choice? [default: Haskell2010]
Add informative comments to each field in the cabal file (y/n)? [default: n] y

Guessing dependencies...

Generating LICENSE...
Generating Setup.hs...
Generating CHANGELOG.md...
Generating src/MyLib.hs...
Generating app/Main.hs...
Generating test/MyLibTest.hs...
Generating my-app.cabal...

Warning: no synopsis given. You should edit the .cabal file and add one.
You may want to edit the .cabal file and add a Description field.

[hs:my-app]>

Please ignore the following warning:

Warning: The package list for 'hackage.haskell.org' does not exist. Run 'cabal
update' to download it.

Nix should take care of handling Haskell libraries not cabal-install. No need to run cabal update.

After this step you should end up with the following set of files:

[hs:my-app]> tree
.
├── CHANGELOG.md
├── LICENSE
├── Setup.hs
├── app
│   └── Main.hs
├── src
│   └── MyLib.hs
├── my-app.cabal
└── test
    └── MyLibTest.hs

3 directories, 7 files

Create a few nix files

The goal of this tutorial is not to make you learn nix because it is a bit complex, but to explain you a bit, nix use a a configuration language and not just a configuration format. So to configure your nix environment you endup writing a nix expression in this nix language. And thus you can call the content of one nix-file in another one for example, or use variables.

The first file to create is the one that will pin the versions of all your packages and libraries:

my-app/nixpkgs.nix ⤓
```
import (fetchTarball https://github.com/NixOS/nixpkgs/archive/19.09.tar.gz) {}
```
The second file is the default.nix file:

my-app/default.nix ⤓
```
{ nixpkgs ? import ./nixpkgs.nix
, compiler ? "default"
, doBenchmark ? false }:
let
  inherit (nixpkgs) pkgs;
  name = "my-app";
  haskellPackages = pkgs.haskellPackages;
  variant = if doBenchmark
            then pkgs.haskell.lib.doBenchmark
            else pkgs.lib.id;
  drv = haskellPackages.callCabal2nix name ./. {};
in
{
  my_project = drv;
  shell = haskellPackages.shellFor {
    # generate hoogle doc
    withHoogle = true;
    packages = p: [drv];
    # packages dependencies (by default haskellPackages)
    buildInputs = with haskellPackages;
      [ hlint
        ghcid
        cabal-install
        cabal2nix
        hindent
        # # if you want to add some system lib like ncurses
        # # you could by writing it like:
        # pkgs.ncurses
      ];
    # nice prompt for the nix-shell
    shellHook = ''
     export PS1="\n\[[${name}:\033[1;32m\]\W\[\033[0m\]]> "
  '';
  };
}
```
It uses the nixpkgs.nix file. But also you can configure it to enable/disable benchmarks while building your application. I do not expect you to understand what is really going on here, but a short explanation is this file take cares of:

    use the pinned version of nixpkgs and should provide a working set of haskell libraries.
    read you .cabal file and find the set of libraries you depends on so nix will be able to download them.
    download a few useful packages for Haskell development, in particular hlint, ghcid, cabal-install, cabal2nix and hindent. I will talk about those tools later.
    take care of handling the nix-shell prompt so you should see the name of your project.

The only things you should manipulate for a new fresh project should be the name and perhaps the buildInputs list to add a few more libraries that could be either Haskell libraries or any library nix know about (for example ncurses, in that case you should write it pkgs.ncurses).

The two last file simply use the default.nix file:

The shell.nix file:

my-app/shell.nix ⤓
```
(import ./. {}).shell
```
And release.nix:

my-app/release.nix ⤓
```
let
  def = import ./. {};
in
 { my_project = def.my_project; }
```
So download those files as well as this .gitignore file:

my-app/.gitignore ⤓
```
dist-newstyle/
result
```
Checking your environment

Now you should see those files in your project:

[hs:my-app]> tree
.
├── CHANGELOG.md
├── LICENSE
├── Setup.hs
├── app
│   └── Main.hs
├── default.nix
├── src
│   └── MyLib.hs
├── my-app.cabal
├── nixpkgs.nix
├── release.nix
├── shell.nix
└── test
    └── MyLibTest.hs

3 directories, 11 files

You shall now enter nix-shell again, but in your my-app directory this time.

[hs:my-app]> nix-shell
warning: Nix search path entry '/nix/var/nix/profiles/per-user/root/channels' does not exist, ignoring
building '/nix/store/j3hi4wm9996wfga61arc2917klfgspwr-cabal2nix-my-app.drv'...
installing
warning: Nix search path entry '/nix/var/nix/profiles/per-user/root/channels/nixpkgs' does not exist, ignoring
warning: file 'nixpkgs' was not found in the Nix search path (add it using $NIX_PATH or -I), at (string):1:9; will use bash from your environment

[my-app:my-app]> which ghcid
/nix/store/ckps9wgbmpckxdvs42p6sqz64dfqiv35-ghcid-0.7.5-bin/bin/ghcid

[my-app:my-app]> cabal run my-app
Build profile: -w ghc-8.6.5 -O1
In order, the following will be built (use -v for more details):
 - my-app-0.1.0.0 (src) (first run)
 - my-app-0.1.0.0 (exe:my-app) (first run)
Configuring library for my-app-0.1.0.0..
Preprocessing library for my-app-0.1.0.0..
Building library for my-app-0.1.0.0..
[1 of 1] Compiling MyLib            ( src/MyLib.hs, /Users/y/hsenv/my-app/dist-newstyle/build/x86_64-osx/ghc-8.6.5/my-app-0.1.0.0/build/MyLib.o )
Configuring executable 'my-app' for my-app-0.1.0.0..
Preprocessing executable 'my-app' for my-app-0.1.0.0..
Building executable 'my-app' for my-app-0.1.0.0..
[1 of 1] Compiling Main             ( app/Main.hs, /Users/y/hsenv/my-app/dist-newstyle/build/x86_64-osx/ghc-8.6.5/my-app-0.1.0.0/x/my-app/build/my-app/my-app-tmp/Main.o )
Linking /Users/y/hs-env/my-app/dist-newstyle/build/x86_64-osx/ghc-8.6.5/my-app-0.1.0.0/x/my-app/build/my-app/my-app ...
Hello, Haskell!
someFunc

Great! It works! Try to run it again:

[my-app:my-app]> cabal run my-app
Up to date
Hello, Haskell!
someFunc

This time, the compilation is not done again. cabal is smart enough not to repeat the compilation again.

You could also use nix-build to compile your app. I think this is nice to do for releases. But for development, you should use cabal.
Add a library

tl;dr: do not be afraid by the lenght of this section in fact, this is straightforward. I just take a lot of time to go through all intermediate steps.

    add the library in the build-depends inside your .cabal file.
    restart nix-shell to download the new dependencies.

If you open the my-app.cabal file in an editor you should see a library section and and executable my-app section. In particular for each section you can see a build-depends sub-section as this one:

...
library
  ...
  build-depends:       base ^>=4.12.0.0
  ...
executable my-app
  ...
  build-depends:       base ^>=4.12.0.0, my-app
  ...

The ^>=4.12.0.0 means that it should use the latest non breaking version of the haskell package base. The author of the base package are responsible not to break the API for minor releases. Haskell libs uses a 4 number versionning quite similar to the semantic versionning scheme with just another minor number for non visible changes. I will not argue much, but mainly, semantic versionning and Haskell versionning are just a "right to break things to your users".

I don't want to talk a lot more about this, but, it would be nice if more people would watch this talk8 related to versionning.

Add the protolude lib in the library build-depends like this:

...
library
  ...
  build-depends:       base ^>=4.12.0.0,
                       protolude
  ...
executable my-app
  ...
  build-depends:       base ^>=4.12.0.0, my-app
  ...

I did not include a version constraint here. This is ok if you do not deploy your library publicly. This would be absolutely awful if you deploy your library publicly. So while developing a private app nobody can see except you, nothing is wrong with this. But I would encourage you to write those version bounds. It is sane to do that, but be warned that your lib might rot if you want it to be part of a working set of libs. So you might be pinged time to time to update some bounds or to adap your code to the breaking change of a lib you are using. Do not think too much about this. This is generally quite trivial work to do to maintain your lib into a working lib set.

Now that you have added protolude modify slightly the code of your app to use it. Change the code inside src/MyLib.hs:

my-app/src/MyLib.hs ⤓

{-# LANGUAGE NoImplicitPrelude #-}
{-# LANGUAGE OverloadedStrings #-}
module MyLib (someFunc) where

import Protolude

someFunc :: IO ()
someFunc = putText "someFunc"

Please do not try to search right now about what this change is doing. It should work mostly as before. The goal here is just to check that you can use another library easily.

So now you should get out of the nix-shell because nix dependencies changed. Generally just type ^D (Ctrl-d) then launch nix-shell --pure.

[my-app:my-app]> cabal build
Warning: The package list for 'hackage.haskell.org' does not exist. Run 'cabal
update' to download it.
Resolving dependencies...
cabal: Could not resolve dependencies:
[__0] trying: my-app-0.1.0.0 (user goal)
[__1] unknown package: protolude (dependency of my-app)
[__1] fail (backjumping, conflict set: my-app, protolude)
After searching the rest of the dependency tree exhaustively, these were the
goals I've had most trouble fulfilling: my-app, protolude


[my-app:my-app]> exit

[hs:my-app]> nix-shell
warning: Nix search path entry '/nix/var/nix/profiles/per-user/root/channels' does not exist, ignoring
building '/nix/store/sr4838rnmzn30j3qc5ray4i2n6n0p8pq-cabal2nix-my-app.drv'...
installing

[my-app:my-app]> cabal build
Build profile: -w ghc-8.6.5 -O1
In order, the following will be built (use -v for more details):
 - my-app-0.1.0.0 (lib) (file src/MyLib.hs changed)
 - my-app-0.1.0.0 (exe:my-app) (configuration changed)
Preprocessing library for my-app-0.1.0.0..
Building library for my-app-0.1.0.0..
[1 of 1] Compiling MyLib            ( src/MyLib.hs, .../my-app/dist-newstyle/build/x86_64-osx/ghc-8.6.5/my-app-0.1.0.0/build/MyLib.o )
Configuring executable 'my-app' for my-app-0.1.0.0..
Preprocessing executable 'my-app' for my-app-0.1.0.0..
Building executable 'my-app' for my-app-0.1.0.0..
[1 of 1] Compiling Main             ( app/Main.hs, .../my-app/dist-newstyle/build/x86_64-osx/ghc-8.6.5/my-app-0.1.0.0/x/my-app/build/my-app/my-app-tmp/Main.o ) [MyLib changed]
Linking .../my-app/dist-newstyle/build/x86_64-osx/ghc-8.6.5/my-app-0.1.0.0/x/my-app/build/my-app/my-app ...

[my-app:my-app]> cabal run my-app
Up to date
Hello, Haskell!
someFunc

Yes!
