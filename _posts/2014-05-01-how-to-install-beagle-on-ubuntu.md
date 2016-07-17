---
layout: single
title: How to install BEAGLE on Ubuntu 14.04 (and make it work with BEAST)
date: 2014-05-01 17:13:23.000000000 -04:00
type: post
published: true
status: publish
categories:
- Hacking
tags:
- phylogenetics
meta:
  _edit_last: '9'
  _syntaxhighlighter_encoded: '1'
  _wpas_done_all: '1'
  _wpas_skip_6113480: '1'
  _wpcom_is_markdown: '1'
author:
  login: francois
  email: francois.michonneau@gmail.com
  display_name: François
  first_name: François
  last_name: Michonneau
---

<a href="https://code.google.com/p/beagle-lib" title="BEAGLE website"
target="_blank">BEAGLE</a> is a library that computes efficiently the
likelihoods needed during maximum-likelihood or Bayesian phylogenetic tree
estimation. It is used by MrBayes, Garli, PhyML and BEAST.

If you are using BEAST on a modern computer, you want to use BEAGLE to take
  advantage of the many cores of your CPU. It will greatly speed up your
  analytical time. BEAGLE can also be
  used to take advantage of your GPU (the processor on your graphic card) but
  I'm not going to talk about this here.


There are some installation instructions on the BEAGLE website but, at least
with my setup it is not quite enough to make it work with BEAST with my setup
(ubuntu 14.04, BEAST 1.8.0 and BEAST 2.1.2).

First, you need to get the necessary dependencies:

```bash
sudo apt-get install build-essential autoconf automake \
  libtool subversion pkg-config openjdk-7-jdk
```

Second, you need to compile and install the library:

Download the latest version of BEAGLE using subversion:

```bash
svn checkout http://beagle-lib.googlecode.com/svn/trunk/ \
   beagle-lib
```

Go to the directory and run the <code>autogen.sh</code> script:

```bash
cd beagle-lib
./autogen.sh
```

Run <code>configure</code>. On the BEAGLE website, the author advise to install
BEAGLE in your home directory, but that makes things more complicated down the
road. If you have root privileges, install it directly where it should go
in <code>/usr/local</code>.


```bash
./configure --prefix=/usr/local
sudo make install
```

At the end of configure, you will get warnings indicating that the openCL and
CUDA libraries were not found and will not be used. Unless you want to use your
GPU this is not a problem. If you want to use your GPU you can specify the path
of the libraries with the arguments <code>--with-opencl=</code> --
				--and <code>--with-cuda=</code>. If you can find
recent packaged (installed with apt-get) versions of these libraries that work
for your GPU, they will be installed in <code>/usr/lib/x86_64-linux-gnu/</code>
(on a 64-bit architecture), so you would need to use: <code>./configure
  --prefix=/usr/local --with-opencl=/usr/lib/x86_64-linux-gnu</code>.


To make sure that the installation was successful, run the checks provided with BEAGLE (this should still be run in the <code>beagle-lib</code> directory):

```bash
make check
```

You should see the tests being passed:

```
=========================================
Testsuite summary for libhmsbeagle 2.1.2
=========================================
# TOTAL: 1
# PASS:  1
# SKIP:  0
# XFAIL: 0
# FAIL:  0
# XPASS: 0
# ERROR: 0
=========================================
```


If all goes well, you can now check that BEAST can use BEAGLE by typing:

```bash
./beast -beagle_info
```

which should show something like this:

````
Using BEAGLE library v2.1.2 for accelerated, parallel likelihood evaluation
2009-2013, BEAGLE Working Group - http://beagle-lib.googlecode.com/
Citation: Ayres et al (2012) Systematic Biology 61: 170-173 | doi:10.1093/sysbio/syr100
BEAGLE resources available:
0 : CPU
    Flags: PRECISION_SINGLE PRECISION_DOUBLE COMPUTATION_SYNCH EIGEN_REAL EIGEN_COMPLEX SCALING_MANUAL SCALING_AUTO SCALING_ALWAYS SCALERS_RAW SCALERS_LOG VECTOR_SSE VECTOR_NONE THREADING_NONE PROCESSOR_CPU FRAMEWORK_CPU
```


So next time you can use some or all of the cores (in the example below 8) of your CPU when running BEAST with the options:

```bash
./beast -beagle -beagle_SSE -threads 8 input.xml
```

If you don't have root privileges and need to install BEAGLE somewhere else
(let's say your home directory using <code>./configure --prefix=$HOME</code>),
then you need to edit the <code>beast</code> executable to include this path in
the <code>java.library.path</code>. In our example the library will be installed
in <code>$HOME/lib</code>, which means that in the executable,
this <code>-Djava.library.path="$BEAST_LIB:/usr/local/lib"</code> needs to be
changed
to <code>-Djava.library.path="$BEAST_LIB:/usr/local/lib:$HOME/lib"</code>. Otherwise,
you are going to see this message:


```
Failed to load BEAGLE library: no hmsbeagle-jni in java.library.path
BEAGLE not installed/found
```

when trying to use BEAGLE with BEAST.
