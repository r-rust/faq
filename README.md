# Publishing R packages with Rust code on CRAN

As of November 2022, CRAN has rustc toolchains available on the platforms that it enforces. Some tips to publish your package on CRAN (or elsewhere).

## Does CRAN allow packages that use rust / cargo?

It is a bit complicated, because CRAN policy is designed for R packages with C/C++/Fortran code that link to libs available on Linux via e.g. apt/yum. However Rust has its own package ecosystem "cargo", which creates a bit of ambiguity. But generally CRAN seems not opposed to packages with Rust code, if they follow the rules.

In recent discussions on the mailing list, the MacOS maintainer states that [CRAN servers have rustc compilers, and packages should make sure to detect them properly](https://stat.ethz.ch/pipermail/r-package-devel/2022q4/008638.html). Later in the thread he also makes it clear that the CRAN maintainers [have themselves no interest in Rust](https://stat.ethz.ch/pipermail/r-package-devel/2022q4/008640.html), and it is up to the community to establish best practices.


## How to setup the package

The most simple example is the [hellorust](https://cran.r-project.org/package=hellorust) package: a template with a minimal setup to call Rust code from R without any frameworks. This includes basic examples of importing cargo dependencies, spawning threads and passing numbers or strings from Rust to R. The [GitHub repository](https://github.com/r-rust/hellorust) shows how to configure CI. A real world use case of this approach is the [gifski](https://cran.r-project.org/package=gifski) package (also on CRAN).

More complex examples are available in the [extendr](https://github.com/extendr) project. The benefit of extendr is that it provides a [rust api](https://crates.io/crates/extendr-api) to work with R objects in the rust code, and a framework to automatically generate bindings. The [helloextendr](https://github.com/extendr/helloextendr) R package shows how to use it.


## Which cargo / rustc version to target?

Try to make your package work with a range of rustc versions. Sometimes this means that you need to pin older versions of dependencies in your `Cargo.lock` file.

On Linux, CRAN uses cargo/rustc versions that ship with the OS distribution, which is usually not the very latest release. On __Debian__, CRAN uses the debian-testing branch and you can find version info here:

 - https://packages.debian.org/testing/cargo
 - https://packages.debian.org/testing/rustc

The current version of cargo/rustc on __Fedora__ can be found here:

 - https://src.fedoraproject.org/rpms/rust

However, some of your users may actually be running Ubuntu/RHEL LTS platforms, which have even older versions of rust. Remember that many R users are on shared servers and don't have permissions to install/upgrade rustc. Hence try to be convervative in assuming the latest dependencies / rust features in your package.

## They say Rust is installed, but `cargo` cannot be found in `PATH`?

This might happen on UNIX-alike platforms. `PATH` is typically configured in the shell's startup scripts like `.bash_profile` and `.zprofile`, but it seems these scripts might not be loaded in some cases. In order to address this, you should either add `$HOME/.cargo/bin` to `PATH` ([example](https://github.com/r-rust/hellorust/blob/8902d6677d70d91b7336a90ba3d8d41f4a9011cd/src/Makevars#L17)) or source `$HOME/.cargo/env`, if it's available, before executing `cargo` command. 

## How to avoid writing in HOME

CRAN has a policy:

> packages should not write in the user’s home filespace (including clipboards), nor anywhere else on the file system apart from the R session’s temporary directory

On most systems, the default cargo configuration keeps a cache in `~/.cargo`, which can be changed by setting the `CARGO_HOME` envvar when invoking cargo. The hellorust package does this in the [`Makevars`](https://github.com/r-rust/hellorust/blob/master/src/Makevars) file.

Personally I think it would be better if CRAN could set a global `CARGO_HOME` on their end, because it is actually not the R package, but the local cargo config that causes writing in the user home dir. However Kurt did not agree, so we set CARGO_HOME in the R package to comply with the policy.

## Why do CRAN checks fail on R-4.1.3 on Windows (r-windows-oldrel) ?

The R for Windows toolchains since R-3.3.0 have been compatible with the rustc `pc-windows-gnu` target. However unfortunately the CRAN infrastructure uses a very old Windows 2008 server (which does not support Rust) to build R packages for R 4.1.3 on Windows. Therefore, packages with Rust code fail to build on CRAN r-win-oldrel. Fortunately CRAN does not enforce r-oldrel, so you may ignore the failure. This problem will automatically disappear after the release of R-4.3.0 in 2023.

Note that you can build and check packages yourself on Windows with R-4.1.x (and older) using e.g. GitHub actions CI or [r-runiverse](https://r-rust.r-universe.dev/ui#builds), because the toolchains are actually fine. The problem is just the CRAN server.


## Should I mention authors of 3rd party cargo crates in the description/license?

Generally speaking, the R package description and license files (that are included with every source package) should declare authorship and copyright of __all source code that is contained within this source package__. Hence, if you copied any material authored by other people in a package that you submit to CRAN, you must mention those people and the copyright in the description, because CRAN hosts this material on their servers.

I don't think the package description should name authors of 3rd party things which your package depends on, or links to, or downloads, if that source code is not included with the source package that is on hosted CRAN. This is not a legal statement of any kind, but simply about labeling what is actually in the box.
