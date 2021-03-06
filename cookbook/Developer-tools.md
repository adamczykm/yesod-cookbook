The Yesod team uses (and has built) quite a number of tools to make our jobs easier. This page is meant to give a comprehensive list of the tools available and the problems they solve. It is not intended to give a deep explanation of how to use them.

## Build tools
* Obviously GHC is our most important tool. These days, we recommend users- and therefore developers- to start off directly with the [Haskell Platform](http://hackage.haskell.org/platform/). Yesod is always officially supported on the current Haskell Platform, though in reality we try to make it work with the previous HP release and any release candidates of GHC.
* [Cabal](http://www.haskell.org/cabal/) is the standard Haskell build system. It's a combination of a library and a command line tool (known as cabal-install, though the executable is just cabal). All packages in the Yesod project are cabalized (i.e., they are built via cabal).
* [[ghci]] for fast re-loading
* [cabal-meta](http://hackage.haskell.org/package/cabal-meta) is another cabal wrapper, but helps with setting up simultaneous builds. This can help prevent dependency hell in many circumstances. It is also compatible with cabal-dev.
* [cabal-src](https://github.com/yesodweb/cabal-src/blob/master/README.md) essentially deploys a package to a local repo (you could think of it as a local alternative to Hackage). This is used internally by the Yesod build scripts. It's very convenient when working on a large number of updated dependencies that have not yet been released to Hackage.
* [mega-sdist](https://github.com/snoyberg/mega-sdist#readme) helps deal with mega-repos, or repositories containing a number of packages as subfolders. It will automatically create tarballs of each package and compare it to the version available on Hackage, letting you know if you need to bump any version numbers.
* [[Virtual machine using VirtualBox and Vagrant|Setting-up-a-virtual-machine,-using-VirtualBox-and-Vagrant]]
* [packdeps](http://hackage.haskell.org/package/packdeps) (and [website](http://packdeps.haskellers.com/)) lets you know when dependencies on a package are out-of-date.

## Text editors
* [TextWrangler](http://www.barebones.com/products/TextWrangler/) is a free code editor for Mac.
    * [BBEdit Codeless Language Module for Haskell](http://code.google.com/p/bbedit-haskell/) can be used as syntax coloring script