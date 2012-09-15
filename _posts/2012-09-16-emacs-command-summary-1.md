---
layout: post
title: "emacs command summary 1"
description: "learning emacs"
category: learning
tags: [emacs]
---
{% include JB/setup %}

{% highlight java %}
C-SP     set-mark-command		 C-q      quoted-insert
C-a      beginning-of-line		 C-r      isearch-backward
C-b      backward-char			 C-s      isearch-forward
C-c      exit-recursive-edit		 C-t      transpose-chars
C-d      delete-char			 C-u      universal-argument
C-e      end-of-line			 C-v      scroll-up
C-f      forward-char			 C-w      kill-region
C-h      help-command			 C-x      Control-X-prefix
TAB      indent-for-tab-command		 C-y      yank
LFD      newline-and-indent		 C-z      suspend-emacs
C-k      kill-line			 ESC      ESC-prefix
C-l      recenter			 C-]      abort-recursive-edit
RET      newline			 C-_      undo
C-n      next-line			 SPC .. ~        self-insert-command
C-o      open-line			 DEL      delete-backward-char
C-p      previous-line

C-h v    describe-variable		 C-h d    describe-function
C-h w    where-is			 C-h k    describe-key
C-h t    help-with-tutorial		 C-h c    describe-key-briefly
C-h s    describe-syntax		 C-h b    describe-bindings
C-h n    view-emacs-news		 C-h a    command-apropos
C-h C-n  view-emacs-news		 C-h C-d  describe-distribution
C-h m    describe-mode			 C-h C-c  describe-copying
C-h l    view-lossage			 C-h ?    help-for-help
C-h i    info				 C-h C-h  help-for-help
C-h f    describe-function

C-x C-a  add-mode-abbrev		 C-x 5    split-window-horizontally
C-x C-b  list-buffers			 C-x ;    set-comment-column
C-x C-c  save-buffers-kill-emacs	 C-x <    scroll-left
C-x C-d  list-directory			 C-x =    what-cursor-position
C-x C-e  eval-last-sexp			 C-x >    scroll-right
C-x C-f  find-file			 C-x [    backward-page
C-x C-h  inverse-add-mode-abbrev	 C-x ]    forward-page
C-x TAB  indent-rigidly			 C-x ^    enlarge-window
C-x C-l  downcase-region		 C-x `    next-error
C-x C-n  set-goal-column		 C-x a    append-to-buffer
C-x C-o  delete-blank-lines		 C-x b    switch-to-buffer
C-x C-p  mark-page			 C-x d    dired
C-x C-q  toggle-read-only		 C-x e    call-last-kbd-macro
C-x C-r  find-file-read-only		 C-x f    set-fill-column
C-x C-s  save-buffer			 C-x g    insert-register
C-x C-t  transpose-lines		 C-x h    mark-whole-buffer
C-x C-u  upcase-region			 C-x i    insert-file
C-x C-v  find-alternate-file		 C-x j    register-to-dot
C-x C-w  write-file			 C-x k    kill-buffer
C-x C-x  exchange-dot-and-mark		 C-x l    count-lines-page
C-x C-z  suspend-emacs			 C-x m    mail
C-x ESC  repeat-complex-command		 C-x n    narrow-to-region
C-x $    set-selective-display		 C-x o    other-window
C-x (    start-kbd-macro		 C-x p    narrow-to-page
C-x )    end-kbd-macro			 C-x q    kbd-macro-query
C-x +    add-global-abbrev		 C-x r    copy-rectangle-to-register
C-x -    inverse-add-global-abbrev	 C-x s    save-some-buffers
C-x .    set-fill-prefix		 C-x u    advertised-undo
C-x /    dot-to-register		 C-x w    widen
C-x 0    delete-window			 C-x x    copy-to-register
C-x 1    delete-other-windows		 C-x {    shrink-window-horizontally
C-x 2    split-window-vertically	 C-x }    enlarge-window-horizontally
C-x 4    ctl-x-4-prefix			 C-x DEL  backward-kill-sentence

ESC C-SP mark-sexp			 ESC =    count-lines-region
ESC C-a  beginning-of-defun		 ESC >    end-of-buffer
ESC C-b  backward-sexp			 ESC @    mark-word
ESC C-c  exit-recursive-edit		 ESC O    ??
ESC C-d  down-list			 ESC [    backward-paragraph
ESC C-e  end-of-defun			 ESC \    delete-horizontal-space
ESC C-f  forward-sexp			 ESC ]    forward-paragraph
ESC C-h  mark-defun			 ESC ^    delete-indentation
ESC LFD  indent-new-comment-line	 ESC a    backward-sentence
ESC C-k  kill-sexp			 ESC b    backward-word
ESC C-n  forward-list			 ESC c    capitalize-word
ESC C-o  split-line			 ESC d    kill-word
ESC C-p  backward-list			 ESC e    forward-sentence
ESC C-s  isearch-forward-regexp		 ESC f    forward-word
ESC C-t  transpose-sexps		 ESC g    fill-region
ESC C-u  backward-up-list		 ESC h    mark-paragraph
ESC C-v  scroll-other-window		 ESC i    tab-to-tab-stop
ESC C-w  append-next-kill		 ESC j    indent-new-comment-line
ESC ESC  ??				 ESC k    kill-sentence
ESC C-\  indent-region			 ESC l    downcase-word
ESC SPC  just-one-space			 ESC m    back-to-indentation
ESC !    shell-command			 ESC q    fill-paragraph
ESC $    spell-word			 ESC r    move-to-window-line
ESC %    query-replace			 ESC t    transpose-words
ESC '    abbrev-prefix-mark		 ESC u    upcase-word
ESC (    insert-parentheses		 ESC v    scroll-down
ESC )    move-past-close-and-reindent	 ESC w    copy-region-as-kill
ESC ,    tags-loop-continue		 ESC x    execute-extended-command
ESC -    negative-argument		 ESC y    yank-pop
ESC .    find-tag			 ESC z    zap-to-char
ESC 0 .. ESC 9  digit-argument		 ESC |    shell-command-on-region
ESC ;    indent-for-comment		 ESC ~    not-modified
ESC <    beginning-of-buffer		 ESC DEL  backward-kill-word

C-x 4 C-f       find-file-other-window	 C-x 4 d  dired-other-window
C-x 4 .  find-tag-other-window		 C-x 4 f  find-file-other-window
C-x 4 b  pop-to-buffer			 C-x 4 m  mail-other-window
{% endhighlight %}
