---
title: Vim folding in Emacs
date: 2014-10-05
tags:
- vim
- emacs
- code
summary: Replicating vim code folding inside Emacs.
categories:
- Tips and Tricks
toc: false
---

## Vim code folding

In Vim you can mark region of text to be hidden (*folded*) with markers like `{{{` and `}}}`, for example:

    # Begin folded region {{{
    ... code ...
    # }}}

I use this type of folding inside my shell init file instead of splitting it for readability; it's easier to just copy one file around instead of multiple files when you want to use your shell configuration on a new server.

## Getting it inside Emacs

In Emacs (24.3.1) this can be achieved using `outline-mode`, but a little bit of *elisp* is needed to achieve a behavior similar to vim's. A code snippet found in the [Emacs wiki](http://www.emacswiki.org/emacs/OutlineMinorMode#toc8) let you specify a *folding marker* similar to vim, but it expect a **level** of folding, like `{{{1`.

I adapted the code snippet to make the **level** optional, defaulting to `0`.

``` elisp
;; code folding with vim compatibility
;; https://raw.githubusercontent.com/yyetim/emacs-configuration/master/elisp/vim-fold.el
(defun set-vim-foldmarker (fmr)
  "Set Vim-type foldmarkers for the current buffer"
  (interactive "sSet local Vim foldmarker: ")
  (if (equal fmr "")
      (message "Abort")
    (setq fmr (regexp-quote fmr))
    (set (make-local-variable 'outline-regexp)
         (concat ".*" fmr "\\([0-9]+\\)?"))
    (set (make-local-variable 'outline-level)
         `(lambda ()
            (save-excursion
              (re-search-forward
               ,(concat fmr "\\([0-9]+\\)") nil t)
              (if (match-string 1)
                  (string-to-number (match-string 1))
                (string-to-number "0")))))))
(global-set-key (kbd "C-<tab>") 'outline-toggle-children)
```
