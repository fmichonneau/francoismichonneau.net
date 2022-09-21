---
layout: single
title: How to do UBSAN and Valgrind tests on a R package that includes C/C++ code
  before submitting to CRAN
date: 2014-03-23 19:59:22.000000000 -04:00
type: post
published: true
status: publish
categories:
- Hacking
tags:
- R
---

After a trying to submit <a
href="https://cran.r-project.org/web/packages/phylobase/index.html">phylobase</a>
to CRAN, I learned a lot about the quality checks that go into a package before
being available to the public. Beyond the typical checks you should perform
routinely during the development of your package, CRAN maintainers also check
for "Memory access" in your C/C++ code. There are some indications on how to do
this in the "Writing R Extensions" <a
href="https://cran.r-project.org/doc/manuals/r-devel/R-exts.html#Checking-memory-access">manual</a>,
but if you haven't done this before it can be a little overwhelming. I thought
it would be useful to have a guide of the things you might want to check for
yourself before submitting your package to CRAN, and be able to reproduce the
issues CRAN maintainers might detect on your package.

To run the memory addressing tests, you need to compile R with special
flags. And, given that you have to compile R yourself to do this, you might as
well use the latest development version, so you can check your code against it
at the same time. While you are it, it will be useful to use a different
compiler than the one that your "normal" R installation has so you can see how
your code behaves. For instance, in my package, I didn't have any C++ dialect
defined, but on CRAN, the r-devel-linux-x86_64-debian-clang flavor used the flag
c++11 which raised some warnings I didn't know about before submitting to
CRAN.

The complete list of flavors used by CRAN to check packages is listed here:
<a href="https://cran.r-project.org/web/checks/check_flavors.html" title="CRAN
check flavors">https://cran.r-project.org/web/checks/check_flavors.html</a>. As
initially some issues were raised on Debian Testing with my package, (and given
that I'm most familiar with Debian as I use Ubuntu), I did my testing using this
flavor, but you could choose something else used by CRAN.

<h2>1. Install Debian in your Virtual Machine (I use VirtualBox for this).</h2>

Make sure you provide enough hard drive space to / during the installation
(>4 Gb), as all the dependencies needed to compile R add up pretty quickly. You
probably don't need to install a desktop environment. Install <code>sudo</code>,
and add the user you created (below `<username>`) to the list of
sudoers.

```bash
su root
apt-get install sudo
sudo adduser <username> sudo
su <username>
```

<h2>2. Switch to Debian testing.</h2>

Update your /etc/apt/sources.list (replace the Debian version name
--e.g. wheezy-- with "testing"), and run


```bash
sudo apt-get update
sudo apt-get upgrade
```

<h2>3. Install needed software</h2>

```bash
sudo apt-get install valgrind subversion r-base-dev \\
     clang-3.4 texlive-fonts-extra texlive-latex-extra
```

From what I understand, the older version of <code>clang</code>, which are the
default even on Debian Testing as I write this, cannot deal with
<code>-fsanitize=undefined</code> for packages.

<h2>4. Get latest R-devel</h2>

```bash
svn co https://svn.r-project.org/R/trunk ~/R-devel
```

<h2>5. Configure the compilation</h2>

Edit the the config.site file uncomment and choose the right options, I have
successfully used:

```bash
CC="clang -std=gnu99 -fsanitize=undefined"
CFLAGS="-fno-omit-frame-pointer -Wall -pedantic -mtune=native"
F77="gfortran"
LIBnn="lib64"
LDFLAGS="-L/usr/local/lib64 -L/usr/local/lib"
CXX="clang++ -std=c++11 -fsanitize=undefined"
CXXFLAGS="-fno-omit-frame-pointer -Wall -pedantic -mtune=native"
FC=${F77}
```

Otherwise, go ahead and compile R-devel

```bash
cd R-devel
./configure --with-x=no --without-recommended-packages
make
sudo make install
```

If it's not the first time you are doing this:

```bash
make clean
```

JAVA is missing so <code>make</code> complains about it, but unless your package needs JAVA it should be fine.

<h2>6. Create a <code>Makevars</code> file</h2>

Create a <code>Makevars</code> file in <code>~/.R/Makevars</code> that contains:

```bash
CC = clang -std=gnu99 -fsanitize=undefined -fno-omit-frame-pointer
CXX = clang -fsanitize=undefined -fno-omit-frame-pointer
```

<h2>7. Test your package</h2>

At this stage, it should all work fine, and you can check your package using
the "Undefined Behaviour Sanitizer". If your package is called "yourPackage":

```bash
R CMD build yourPackage/
R CMD check yourPackage_0.0.1.tar.gz
```

<h2>8. Check the results</h2>

If the compilation goes without any issues, the check should perform as
usual.

Make sure you see something like: <code>R Under development (unstable)
(2014-03-23 r65264)</code> in your <code>yourPackage.Rcheck/00check.log</code>
(with a recent date and a higher revision number to indicate you are actually
using the latest version of R-devel).

If your test files have their <code>*.Rout.save</code> counterpart then,
issues will be listed in the 00check.log. Otherwise, this command should list
where to look for UBSAN issues:


```bash
grep runtime\ error yourPackage.Rcheck/tests/*.Rout -R
```

<h2>9. Run the Valgrind tests</h2>

You will also want to use Valgrind to check for possible problems in your C/C++
code. First, you probably want to use on all your test files (as well as
examples and vignettes) during a regular <code>R CMD check</code>:

```bash
R CMD check --as-cran --use-valgrind yourPackage_0.0.1.tar.gz
```

and while you are debugging your issues, you can run it on a specific file:

```bash
R -d valgrind --vanilla > tests/myTest1.R
```

More details can be found in the <a
href="https://cran.r-project.org/doc/manuals/r-devel/R-exts.html#Using-valgrind">R
manual</a>.

Feedback on this document is welcome.
