---
layout: post
title: VIM can help your OCD
tags:
  - vim
---
I don't know your opinion, but myself, I like, I mean, I really like, when things are clean, well indented, sorted
alphabetically... At some point, just readable.  If I start to work on a code that doesn't follow these rules, I **have**
to change it. Otherwise I'm not able to concentrate of what does the code.

If you have the same needs, you know how painful it is to change this code manually, over and over. So, I will explain you
how we can tweak VIM to take care of our Obsessive Compulsive Disorder.

First example, Ruby. I like when my assignments are perfectly aligned on the `=`. I also like when my hash-es are
perfectly aligned on the `:`. To do alignment, there is already a bunch of good plugins out of there. Basically the one
that fully matches my needs is [EasyAlign](https://github.com/junegunn/vim-easy-align). It handles all the alignments I need,
out of the box (i.e: `=`, `:`, `.`).

Currently, I'm working on an old application (more than 6 years old) and lot of hash-es still use the old syntax
`:my_key => "value"`. So I also want to be able to change these hash-es to follow the new syntax. To do it, we can create
a simple VIM command in our vimrc.

```vim
" ~/.vimrc

" Based on a range, it replace `:my_key => "value"` to `my_key: value`
command! -range OldToNewHash <line1>,<line2>s/:\([a-zA-Z-0-9_]\+\)\s*=>/\1:/g
```

Now, we are able to replace an old hash to a new one, just calling `:OldToNewHash` on a selection. We are also able to
align the keys calling `:EasyAlign :` on the same selection. Cool!

Typing these commands is not hard, but it's slow. To make it faster we can start by mapping these commands to a shortcut.
Here an example:

```vim
" ~/.vimrc
vmap <Leader>t <Plug>(EasyAlign)
vmap <Leader>ro :OldToNewHash<cr>
```

We are now able to align our keys typing `\t:` and transforming our old hash to a new one with `\ro`. Great!

But I'm definitvely too lazy to type two commands, even if there are shotcuts, just to change my hashes, and I guess you
are too. So, we could try to create another command that groups the two other ones.

```vim
" ~/.vimrc
function! s:RubyObsessiveCompulsiveDisorder() range
  exec a:firstline . ',' . a:lastline . 'OldToNewHash'
  exec a:firstline . ',' . a:lastline . 'EasyAlign :'
endfunction

command! -range RubyObsessiveCompulsiveDisorder <line1>,<line2>call s:RubyObsessiveCompulsiveDisorder()
```

Wonderful! We can now transform our old hash-es to new ones and align the keys in one pass. We just have to call
the command `RubyObsessiveCompulsiveDisorder`. Of course, we can add a shortcut to call it even more easily.

```vim
" ~/.vimrc
vmap <Leader>rocd :RubyObsessiveCompulsiveDisorder<cr>
```

All good, we have just fixed our OCD for Ruby. But, what's about SCSS for example? We could want to sort the properties
alphabetically and align the values as well.

We already know how to align the values (thanks to `EasyAlign`), we just have to learn how to sort our properties
alphabetically. Hopefully, VIM comes with lot of built-in commands, like `:sort` that... well... Sort your lines
alphabetically.

So we can follow the same process as previously and create our other OCD method and add an alias for it:

```vim
" ~/.vimrc
function! s:SCSSObsessiveCompulsiveDisorder() range
  exec a:firstline . ',' . a:lastline . 'sort'
  exec a:firstline . ',' . a:lastline . 'EasyAlign :'
endfunction

command! -range SCSSObsessiveCompulsiveDisorder <line1>,<line2>call s:SCSSObsessiveCompulsiveDisorder()

vmap <Leader>socd :SCSSObsessiveCompulsiveDisorder<cr>
```

OK, that's cool but now you could argue it would be hard to remind the name of all the OCD command we have... And I would
be totally agree. Moreover, the Ruby one is useful only in the context of a Ruby file. The second one is handy only in
the context of a SCSS file. So, it would be really neat if we could call the same method or shortcut and have a
behaviour depending of the file type, isn't it? Yes, of course it would!

Let's do that!

Because, VIM is the best editor ever, we just have to move our commands and functions to a file named `ruby` or `scss`
in our `~/.vim/after/ftplugin/` folder. Then, automatically, each time you open a file and VIM detects it's a ruby file,
it will source your `~/.vim/after/ftplugin/ruby.vim` file. It means we can have twice the same command in different
`ftplugin` and we should be able to call the good one each time, depending on the file.

At this point, we could have the expected result, with the following configuration:

```vim
" ~/.vimrc
vmap <Leader>ocd :ObsessiveCompulsiveDisorder<cr>
vmap <Leader>ro :OldToNewHash<cr>
vmap <Leader>t <Plug>(EasyAlign)

" ~/.vim/after/ftplugin/ruby.vim
function! s:ObsessiveCompulsiveDisorder() range
  exec a:firstline . ',' . a:lastline . 'OldToNewHash'
  exec a:firstline . ',' . a:lastline . 'EasyAlign :'
endfunction

command! -range ObsessiveCompulsiveDisorder <line1>,<line2>call s:ObsessiveCompulsiveDisorder()
command! -range OldToNewHash <line1>,<line2>s/:\([a-zA-Z-0-9_]\+\)\s*=>/\1:/g

" ~/.vim/after/ftplugin/scss.vim
function! s:ObsessiveCompulsiveDisorder() range
  exec a:firstline . ',' . a:lastline . 'sort'
  exec a:firstline . ',' . a:lastline . 'EasyAlign :'
endfunction

command! -range ObsessiveCompulsiveDisorder <line1>,<line2>call s:ObsessiveCompulsiveDisorder()
```

Neat! We can now call the `\ocd` in a Ruby file or in a SCSS file and it will apply the good command based on the current
filetype. Below, an example on a old ruby hash.

![OCD for Ruby](/assets/images/vim-can-help-your-ocd/ocd.gif)
