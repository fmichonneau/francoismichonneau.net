---
title: "Using Air to reformat code with Emacs ESS"
excerpt: "A snippet to add to Emacs configuration file to use the new R code formatter Air"
layout: "single"
date: "2025-02-24"
type: "post"
published: true
categories: ["Hacking"]
tags: ["r", "ess"]
---

[Air](https://posit-dev.github.io/air/) is an R formatter and language server
written in Rust.

It is very [fast and opiniated](https://www.tidyverse.org/blog/2025/02/air/). It
integrates with VSCode and Positron, RStudio (and soon with Zed).

Maybe there is a better way at integrating it with Emacs and ESS but for the
time being, I wrote this short snippet that uses its command line interface to
reformat the current buffer on save:

```lisp
;; use Air to format the content of the file
(defun run-air-on-r-save ()
  "Run Air after saving .R files and refresh buffer."
  (when (and (stringp buffer-file-name)
             (string-match "\\.R$" buffer-file-name))
    (let ((current-buffer (current-buffer)))
      (shell-command (concat "air format " buffer-file-name))
      ;; Refresh buffer from disk
      (with-current-buffer current-buffer
        (revert-buffer nil t t)))))

(add-hook 'after-save-hook 'run-air-on-r-save)
```

From my limited testing, it works well enough for now. 