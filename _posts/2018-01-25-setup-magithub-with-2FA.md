---
layout: single
title: "How to setup magithub if you have GitHub 2-factor authentication enabled?"
date: 2018-01-25
type: post
published: true
status: publish
categories: ["Hacking"]
tags: ["emacs", "magit", "magithub"]
excerpt: "The steps involved to create the authinfo.gpg file used by magithub when you have 2FA enabled on GitHub"
---

If you are trying to set up `magithub` when you have 2 factor authentication enabled, here are the steps you need to take:

- Go to https://github.com/settings/tokens and create a personal token, and give
  it the name that the prompt suggest. For me it was: "Emacs package magithub @
  francois-XPS-15-9560", and give it the following scopes: "notification",
  "repo" and "user".
- create a file `~/.authinfo` with the following:
  
  ```
  machine api.github.com login YOUR_GITHUB_USERNAME^magithub password <your token>
  ```
- encrypt the file (assumes you have GPG setup) by running: `M-x epa-encrypt-file` and give it `~/.authinfo`.
- Make sure that `~/.authinfo.gpg` was created and that its content is right.
- Delete the unencrypted `~/.authinfo`
- Do `M-x customize-variable RET auth-sources` and put `~/.autoinfo.gpg` first
  in the list of files inspected.
  
  

