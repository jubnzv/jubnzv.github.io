---
title: "Using (neo)vim for C++ development"
date: 2021-01-08T09:23:31+03:00
toc: true
tags: [cpp, vim, tools]
---

## Background

I've been writing code in vim for the past several years, and in this post I'd like to share some tips on configuring a development environment. This post contains some notes on configuration that would have helped me when I first started using vim and working on my own config. I hope that as a result of reading this, you will be able to improve your workflow with some new features and make the development process easier and more convenient.

In this article, we will look at common tasks that occur when editing code and try to automate and improve them using vim. Each section contains a brief description of the problem, a proposed solution, overview of alternatives, a full code listing for the configuration, and a screenshot or animated screencast with a demonstration. At the end, additional links to useful plugins and resources will be provided.

Most of the tasks come down to installing and properly configuring one or more plugins. I assume that you are an experienced vim user and already use one of the plugin managers.

All of these tips are applicable in both vim and neovim. Also, despite the title, some of these tips can be applied not only to C++, but also to any other language.

## Removing trailing whitespaces

Let's start with a very simple problem. Some code conventions restrict the leaving of trailing whitespaces. For example, the Linux Kernel [prohibits](https://www.kernel.org/doc/html/v4.10/process/coding-style.html#spaces) from doing this.

So we'd like to see when we accidentally leave a space at the end of a line. Let's add the following lines in your vim configuration file:

```vim
highlight ExtraWhitespace ctermbg=red guibg=red
match ExtraWhitespace /\s\+$/
au BufWinEnter * match ExtraWhitespace /\s\+$/
au InsertEnter * match ExtraWhitespace /\s\+\%#\@<!$/
au InsertLeave * match ExtraWhitespace /\s\+$/
au BufWinLeave * call clearmatches()
```

As a result, extra whitespaces will be highlighted as follows:

![](/posts/vim-modern-cpp/trailing-ws.png)

We also want to remove extra spaces by the single key binding. Let's define it:

```vim
" Remove all trailing whitespaces
nnoremap <silent> <leader>rs :let _s=@/ <Bar> :%s/\s\+$//e <Bar> :let @/=_s <Bar> :nohl <Bar> :unlet _s <CR>
```

So we can solve this simple problem.

Alternatives:
* You [can show](https://www.reddit.com/r/vim/comments/82yv3p/anyone_know_how_to_get_the_dots_for_leading/dveznas?utm_source=share&utm_medium=web2x&context=3) all whitespaces and line breaks in the source code.
* There is [editorconfig](https://editorconfig.org/) [plugin](https://github.com/editorconfig/editorconfig-vim) for vim that supports the `trim_trailing_whitespace` option. When it is set to `true` the plugin will remove any whitespace characters preceding newline characters.
* See [clang-format](##formatting-source-code-with-clang-format) section below.

## Accessing Standard Library documentation using `cppman`

Vim ships with a man page viewer that works out of the box on any modern \*nix via [:Man](https://vimhelp.org/filetype.txt.html#ft-man-plugin) command. But the system man pages do not contain the definitions for C++. Let's fix it by installing [Cppman](https://github.com/aitjcize/cppman).

Cppman is a set of C++ 98/11/14/17/20 manual pages for Linux, with source from [cplusplus.com](https://cplusplus.com/) and [cppreference.com](https://cppreference.com). You can easily install using `pip` or the package manager for your distribution according to their [Installation guide](https://github.com/aitjcize/cppman#installation). Next, let's configure vim.

By default, when you press `K`, vim grabs the keyword under the cursor, separated by [iskeyword](https://vimhelp.org/options.txt.html#%27iskeyword%27) symbols. There is one problem. The `:` symbol often found in C++ definitions is not included in the `iskeyword` array. So, for example, when you press `K` under the keyword `std::string`, vim will try to find the man page for `std`, instead of the full identifier. Let's fix it with this small function:

```vim
function! s:JbzCppMan()
    let old_isk = &iskeyword
    setl iskeyword+=:
    let str = expand("<cword>")
    let &l:iskeyword = old_isk
    execute 'Man ' . str
endfunction
command! JbzCppMan :call s:JbzCppMan()
```

We also need to remap the default `K` key binding to use our function in C++:

```vim
au FileType cpp nnoremap <buffer>K :JbzCppMan<CR>
```

Here is a result:

![](/posts/vim-modern-cpp/cppman.png)

## Browsing online documentation from vim

But what if you also want to quickly access online documentation for your favorite library or framework? Or may be search for unknown keywords on StackOverflow, Google or similar sites? And it is desirable to make a solution that can be easily extended to support more search engines.

[open-browser.vim](https://github.com/tyru/open-browser.vim) can help us. This is a plugin that allows you open URLs in your favorite browser using vim.

Install it using your package manager. Then let's add a simple configuration to access cppreference and Qt documentation pages:

```vim
let g:openbrowser_search_engines = extend(
\ get(g:, 'openbrowser_search_engines', {}),
\ {
\   'cppreference': 'https://en.cppreference.com/mwiki/index.php?title=Special%3ASearch&search={query}',
\   'qt': 'https://doc.qt.io/qt-5/search-results.html?q={query}',
\ },
\ 'keep'
\)
```

And add some convenient key bindings by this way:

```vim
nnoremap <silent> <leader>osx :call openbrowser#smart_search(expand('<cword>'), "cppreference")<CR>
nnoremap <silent> <leader>osq :call openbrowser#smart_search(expand('<cword>'), "qt")<CR>
```

As a result we can quickly browse online documentation for the word under the cursor:

![](/posts/vim-modern-cpp/openbrowser.gif)

Further use of this plugin is limited only by your imagination. For example, you can use it to search for similar C++ snippets in open source projects on GitHub:

```vim
'github-cpp': 'http://github.com/search?l=C%2B%2B&q=fork%3Afalse+language%3AC%2B%2B+{query}&type=Code',
```

Or integrate it with [grep.app](https://grep.app/) or [Debian Code Search](https://codesearch.debian.net/). Or add integration with your favorite online translation service, if you work with multiple languages. You get the idea. The main thing is to write the configuration correctly.

Alternatives:
* [zeavim.vim](https://github.com/KabbAmine/zeavim.vim) is a plugin that implements the integration with [Zeal](https://zealdocs.org/) – free and open source offline documentation browser. In fact, it solves the same problem. You may want to use it if you don't have an internet connection, or you need access to documentation for a specific version of the framework that isn't available online.
* [devdocs.vim](https://github.com/rhysd/devdocs.vim) is a plugin for [devdocs](https://devdocs.io/) which also provides multiple API documentations.

## Switching between source and header file

Switching between source and header files is another common operation when working with C++.

After searching and trying many solutions, I settled on [vim-fswitch](https://github.com/derekwyatt/vim-fswitch) plugin. Personally, I like this plugin for its simple configuration for various file types and the absence of third-party dependencies.

Install it with your favorite package manager. It will perfectly works out of the box, but sometimes you want to configure switch destination files like this:

```vim
au BufEnter *.h  let b:fswitchdst = "c,cpp,cc,m"
au BufEnter *.cc let b:fswitchdst = "h,hpp"
```

There is also one more nuance (thanks Elias Daler for [sharing this](http://disq.us/p/2feabqq)!).

A lot of C++ projects follow this convention for splitting headers and sources:

```
include/<project-name>/<path>/some_header.h
src/<path>/some_source.cpp
```

To make FSwitch work with such cases (when switching from header to source), you need to add this to vimrc:

```vim
au BufEnter *.h let b:fswitchdst = 'c,cpp,m,cc' | let b:fswitchlocs = 'reg:|include.*|src/**|'
```

It also will be convenient to set up some key bindings:

```vim
nnoremap <silent> <A-o> :FSHere<cr>
" Extra hotkeys to open header/source in the split
nnoremap <silent> <localleader>oh :FSSplitLeft<cr>
nnoremap <silent> <localleader>oj :FSSplitBelow<cr>
nnoremap <silent> <localleader>ok :FSSplitAbove<cr>
nnoremap <silent> <localleader>ol :FSSplitRight<cr>
```

Alternatives:
+ There are many other options described in this [vimwiki article](https://vim.fandom.com/wiki/Easily_switch_between_source_and_header_file). You can try them or write your own solution.

## Using ctags

`ctags` is a convenient indexing tool that allows you to go to symbol definition from your project using `tags` database. This is an old and reliable tool, that is [recommended](https://kernelnewbies.org/FAQ/CodeBrowsing) [to use](https://stackoverflow.com/a/33682137/8541499) for large code bases like Linux Kernel where other tools like LSP may get stuck.

The most actual and maintained implementation of ctags is [universal-ctags](https://github.com/universal-ctags/ctags). It is available in all popular distributions.

Vim has integrated ctags support. But here is one annoying thing. We need to generate `tags` database for each project using something like `ctags -R .`. Moreover, we need re-generate `tags` when editing the project.

This of course can be automated. Install [vim-guttentags](https://github.com/ludovicchabant/vim-gutentags) is a plugin that will asynchronously (re)generate tag files as you work. I also suggest adding a simple configuration to avoid indexing of some unwanted files:

```vim
set tags=./tags;
let g:gutentags_ctags_exclude_wildignore = 1
let g:gutentags_ctags_exclude = [
  \'node_modules', '_build', 'build', 'CMakeFiles', '.mypy_cache', 'venv',
  \'*.md', '*.tex', '*.css', '*.html', '*.json', '*.xml', '*.xmls', '*.ui']
```

This makes your workflow simpler and more convenient. See also [this](https://vim.fandom.com/wiki/Browsing_programs_with_tags) detailed vimwiki article if you need more information about using ctags.

## Exploring source file structure with `vista.vim`

When you are working with unfamiliar source code, it may be convenient to browse the structure of the current source file. What classes, functions, macroses are defined here?

[vista.vim](https://github.com/liuchengxu/vista.vim) is a plugin that helps us. It opens a split with the current file definitions. Here is what it looks like:

![](/posts/vim-modern-cpp/vista.png)

This plugin shows both LSP and ctags symbols. The LSP configuration is beyond the scope of this post, but it is a very useful feature that can give more accurate information.

To use it, install [vista.vim](https://github.com/liuchengxu/vista.vim) with your favorite package manager and add a convenient key binding to toggle Vista split:

```vim
nnoremap <silent> <A-6> :Vista!!<CR>
```

Alternatives:
* [tagbar](https://github.com/preservim/tagbar) is another plugin that provides a way to get the overview of the file structure. Unlike vista.vim, it doesn't support LSP symbols and doesn't provide useful utility functions for retrieving the information about the symbol under the cursor. But it's simpler and more reliable.

## Display the current function in the status line

Another feature of [vista.vim](https://github.com/liuchengxu/vista.vim) is that it shares information of the symbol under the cursor. Using this we can display the current function name in the status line. This helps us navigate when moving through a large file.

Bellow is an example of configuration for the [lightline](https://github.com/itchyny/lightline.vim), but you can easily adapt it to your status bar:

```vim
function! LightlineCurrentFunctionVista() abort
  let l:method = get(b:, 'vista_nearest_method_or_function', '')
  if l:method != ''
    let l:method = '[' . l:method . ']'
  endif
  return l:method
endfunction
au VimEnter * call vista#RunForNearestMethodOrFunction()
```

Here's what it looks like:

![](/posts/vim-modern-cpp/current-function.gif)

Is you need the complete lightline configuration, see [this snippet](https://github.com/jubnzv/dotfiles/blob/dd46c2d940c13a0db8d94b02934659018358579a/.config/nvim/init.vim#L435-L462) and take a look at lightline documentation.

Alternatives:
* There are many alternatives provided by popular LSP plugins. For example, [lsp-status.nvim](https://github.com/nvim-lua/lsp-status.nvim) can provide the same information for Neovim's built-in LSP server.

## vimspector: Interactive debugging inside vim

One of the interesting features supported by modern text editors and IDEs is the [Debug Adapter Protocol (DAP)](https://microsoft.github.io/debug-adapter-protocol/). The idea behind the Debug Adapter Protocol is to standardize the communication between the development tool and a concrete debugger or runtime:

![](/posts/vim-modern-cpp/DAP.png)

It was originally developed to use with [Visual Studio Code](https://code.visualstudio.com/api/extension-guides/debugger-extension), but later it [was moved](https://code.visualstudio.com/blogs/2018/08/07/debug-adapter-protocol-website) to separate project. Currently, DAP support is implemented in all popular text editors, including vim.

The most popular DAP adapter for vim provided by awesome [vimspector](https://github.com/puremourning/vimspector) plugin.

The installation process is a bit tricky. We will need [vscode-cpptools](https://github.com/microsoft/vscode-cpptools) to make it works with C and C++ executables. Fortunately, vimspector ships with convenient script that automates our installation. The installation process is also [well documented](https://github.com/puremourning/vimspector#install-some-gadgets).

If you are using [vim-plug](https://github.com/junegunn/vim-plug) plugin manager as well as I do, write the following in your configuration file:

```vim
Plug 'puremourning/vimspector', {
  \ 'do': 'python3 install_gadget.py --enable-vscode-cpptools'
  \ }
```

For other plugin managers, the installation will be actually the same. You just need to execute `install_gadget.py` script once after the installation as the [documentation](https://github.com/puremourning/vimspector#install-some-gadgets) says.

Next let's set up some key bindings:

```vim
command! -nargs=+ Vfb call vimspector#AddFunctionBreakpoint(<f-args>)

nnoremap <localleader>gd :call vimspector#Launch()<cr>
nnoremap <localleader>gc :call vimspector#Continue()<cr>
nnoremap <localleader>gs :call vimspector#Stop()<cr>
nnoremap <localleader>gR :call vimspector#Restart()<cr>
nnoremap <localleader>gp :call vimspector#Pause()<cr>
nnoremap <localleader>gb :call vimspector#ToggleBreakpoint()<cr>
nnoremap <localleader>gB :call vimspector#ToggleConditionalBreakpoint()<cr>
nnoremap <localleader>gn :call vimspector#StepOver()<cr>
nnoremap <localleader>gi :call vimspector#StepInto()<cr>
nnoremap <localleader>go :call vimspector#StepOut()<cr>
nnoremap <localleader>gr :call vimspector#RunToCursor()<cr>
```

After that, our setup is ready to go.

For each project we need to create [vimspector.json](https://github.com/puremourning/vimspector#usage) which contains information about how to start a debugger session. This file is pretty simple, and you will usually just copy it between your projects with minimal changes. Here is an example content:

```json
{
  "configurations": {
    "Launch": {
      "adapter": "vscode-cpptools",
      "configuration": {
        "request": "launch",
        "program": "testrunner",
        "externalConsole": true
      }
    }
  }
}
```

Put this file in the project root, add breakpoints and launch the debugger session with defined key binding. Here is a small demo:

![](/posts/vim-modern-cpp/vimspector.gif)

This plugin supports all common debugger actions like step over/into/out, breakpoints, watchlists, etc. It can also be used for [remote debugging](https://github.com/puremourning/vimspector#remote-debugging), which is very useful in embedded area.

Alternatives:
* Neovim users can be also interested in [nvim-dap](https://github.com/mfussenegger/nvim-dap) – DAP client written in Lua.

## Snippets and print-statement-debugging

But sometimes you can't properly configure your project to use the debugger. Often, this problem occurs in embedded area or when you work with large code base and you don't have debugging information. In this case, you can use the rudimentary print-statement-debugging method. And the snippets will help you perfectly in this case.

[Snippets](https://en.wikipedia.org/wiki/Snippet_(programming)) are smart templates that will insert text for you and adapt it to their context. For those who have never used them, it will be easier to show a small demo:

![](/posts/vim-modern-cpp/snippets-demo.gif)

Snippets are a powerful tool. If you never use it before, I strongly recommend the article [How I'm able to take notes in mathematics lectures using LaTeX and Vim](https://castel.dev/post/lecture-notes-1/) by Gilles Castel. This is a comprehensive article about note-taking in Mathematics lectures in Vim with good introduction to snippets. It's just fascinating. Some of his advices can be applied to your development workflow.

But let's back to our use-case. We need to install two plugins: [ultisnips](https://github.com/SirVer/ultisnips) and [vim-snippets](https://github.com/honza/vim-snippets).

Open C++ file in vim and execute `:UltiSnipsEdit` to open snippets file. Add the following lines to it:

```vim
snippet prd
std::cout << __PRETTY_FUNCTION__ << " " << $1 << std::endl; // prdbg
endsnippet
```

Save it and execute `:call UltiSnips#RefreshSnippets()` to apply changes.

Next define a helper function:

```vim
function! s:JbzRemoveDebugPrints()
  let save_cursor = getcurpos()
  :g/\/\/\ prdbg$/d
  call setpos('.', save_cursor)
endfunction
command! JbzRemoveDebugPrints call s:JbzRemoveDebugPrints()
```

This function will remove our print statements. We can also define a convenient key binding to call it:

```vim
au FileType c,cpp nnoremap <buffer><leader>rd :JbzRemoveDebugPrints<CR>
```

So, we can very quickly add our debug prints and delete them with a single command. Here is a demo:

![](/posts/vim-modern-cpp/snippets-debug-printing.gif)

Improve your prints, add more snippets and use it. However, do not forget that the use of a debugger is preferable in most cases.

## Formatting source code with `clang-format`

[clang-format](https://clang.llvm.org/docs/ClangFormat.html) is a tool to automatically format C/C++/Java/JavaScript/Objective-C/Protobuf/C# code, so that developers don't need to worry about style issues during code reviews.

It can be simply integrated into vim. Install `clang-format` tool from your distribution repositories and create this function in the vim configuration:

```vim
function! s:JbzClangFormat(first, last)
  let l:winview = winsaveview()
  execute a:first . "," . a:last . "!clang-format"
  call winrestview(l:winview)
endfunction
command! -range=% JbzClangFormat call <sid>JbzClangFormat (<line1>, <line2>)
```

Add some key bindings:

```vim
" Autoformatting with clang-format
au FileType c,cpp nnoremap <buffer><leader>lf :<C-u>JbzClangFormat<CR>
au FileType c,cpp vnoremap <buffer><leader>lf :JbzClangFormat<CR>
```

And let's see how does it work:

![](/posts/vim-modern-cpp/clang-format.gif)

You can also select text regin in visual mode and call `clang-format` with the same key binding.

Alternatives:
+ [neoformat](https://github.com/sbdchd/neoformat) is a universal plugin that can run the arbitrary code formatters. Note that it also can be configured to use [cmake_format](https://github.com/cheshirekow/cmake_format/) `CMakeLists.txt` formatter, which can be handy in C++ projects.
+ [vim-clang-format](https://github.com/rhysd/vim-clang-format) – advanced plugin for running `clang-format`. See: [What is the difference from clang-format.py?](https://github.com/rhysd/vim-clang-format#what-is-the-difference-from-clang-formatpy) in their README.
+ There is also [clang-format.py](https://github.com/llvm/llvm-project/blob/62ec4ac90738a5f2d209ed28c822223e58aaaeb7/clang/tools/clang-format/clang-format.py) script in LLVM distribution. You can use it according to the [documentation](https://clang.llvm.org/docs/ClangFormat.html#vim-integration).

## Enhancing syntax highlighting for modern C++

By default, vim has very poor syntax highlighting for C++. The problem is that C++ cannot be parsed without complete semantic analysis.

For example, if we have `f(x)` expression, depending on context `f` [may be](https://youtu.be/ltCgzYcpFUI?list=PLgS1xfF0v4bEOFCZDgJNasvkLS5Yv9_LI&t=2026):
- a function (or function pointer or function reference)
- an instance of a class overloading `operator()`
- an object implicitly convertible to one of the above
- an overloaded function name (at any of multiple scopes)
- the name of one or more templates (at any of multiple scopes)
- the name of one or more template specializations
- ... several of the above.

We will also need information about all header files and libraries included in the project.

Thus, we definitely need to get semantic information about the context.

One option is to use semantic highlighting provided by the LSP server. Semantic highlighting is an extension of LSP [added](https://github.com/Microsoft/language-server-protocol/issues/18) to the protocol several years ago.

The problem is that using semantic highlighting significantly complicates the configuration of the LSP server and client. You will also need a properly working LSP server for each project, which can be inconvenient if you don't want to run the build to get the `compile_commands.json` file. However, this solution gives you the most correct syntax highlighting. If you want to use it, consider [vim-lsp-cxx-highlight](https://github.com/jackguo380/vim-lsp-cxx-highlight) plugin.

I suggest to take a look at a simpler solution. We can use an extended vim syntax file that contains keywords added in the recent standards and common C++ functions from the Standard Library.

To do this, install [vim-cpp-modern](https://github.com/bfrg/vim-cpp-modern) using your package manager.

Result of plugin activation (on the right):

![](/posts/vim-modern-cpp/cxx-syntax.png)

It should be noted that sometimes this plugin will highlight keywords inaccurately. For example, a variable named `string` used in your code will be colored as a keyword, since the plugin contains the corresponding rule for the `std::string` class. However, this is the simplest and most reliable solution.

Alternatives:
+ [vim-lsp-cxx-highlight](https://github.com/jackguo380/vim-lsp-cxx-highlight) mentioned above
+ Neovim nightly [has](https://github.com/nvim-treesitter/nvim-treesitter) integration with [tree-sitter](https://github.com/tree-sitter/tree-sitter) – an incremental parsing system for programming tools. You can try using it and tell us in the comments how it works in comparison with LSP semantic highlighting and vim syntax files based on the regular expressions.

## Ideas for further configuration

I hope you can get some snippets from my post and simplify your vim-based C++ development environment. Of course, I can't fit all information about vim into one article. Therefore, I tried to write about some useful features that will certainly be useful to most developers.

Some topics deserve a separate writings. However, I consider it necessary to at least list them here:
+ LSP plugins. I intentionally skiped this topic, because there are lots of alternatives: [vim-lsp](https://github.com/prabirshrestha/vim-lsp), [LanguageClient-neovim](https://github.com/autozimu/LanguageClient-neovim), [coc.nvim](https://github.com/neoclide/coc.nvim), [Neovim's built-in LSP server](https://github.com/neovim/nvim-lspconfig), etc. Each of them contains its own configuration features and the additional plugins, therefore, it is impossible to fully cover this in one article.
+ Autocompletion plugins, the choice of which usually depends on your LSP plugin.
+ Take a look at handy plugins that improves the visual appearance: [indentLine](https://github.com/Yggdroot/indentLine), [vim-illuminate](https://github.com/RRethy/vim-illuminate), [rainbow](https://github.com/luochen1990/rainbow).

If you have any additions, comments, and suggestions to improve this post, I would be very grateful if you write about this in the comments.
