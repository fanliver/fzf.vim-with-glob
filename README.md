# [WIP] fzf.vim-with-glob
An instruction about how to customize fzf.vim with ripgrep to support glob

Search text in all files
![image](https://user-images.githubusercontent.com/6322508/118121461-bf357380-b41b-11eb-919c-422e5f2c5000.png)

Search text in non-test file
![image](https://user-images.githubusercontent.com/6322508/118121471-c0ff3700-b41b-11eb-9a62-2d31eac684b7.png)

Search text in test file only
![image](https://user-images.githubusercontent.com/6322508/118121484-c3fa2780-b41b-11eb-88de-05b20231ab36.png)


## Customize fzf reload when typing
To understand about fzf event bindings, please read
[Fzf - Execute external programs](https://github.com/junegunn/fzf#executing-external-programs).

With the `change` event in fzf, you can make your own event bindings to execute
a custom shell command when user is typing in fzf search window.

For example:
```
fzf --bind 'change:execute(echo You are typing {q})'
```
`fzf` will run `echo You are typing {q}` to output what you are typing into the standard output.
Notice that `{q}` is denoted for the query string in `fzf`.
Try typing `abc` and `<ESC>` to exit fzf, you will see:
```
> fzf --bind 'change:execute(echo You are typing {q})'
You are typing a
You are typing ab
You are typing abc
```
Fzf also supports `reload` action to reload the data source of the search window.
Let's try
```
fzf --bind 'change:reload(date +"%T")' --phony
```
Now try pressing any key to see the time is updated in your fzf search window.
The `--phony` flag to ensure that fzf is not doing the search, so the time is not hidden,
fzf just acts as a menu selector.

With the `change` event and the `reload` action, now we can make fzf co-operate with 
`Ripgrep` to support glob with following steps:

1. Write a shell function to do the search which accepts glob
First, write a shell function that take a search pattern including glob.
Inside that shell function, the search pattern is transforms into two parts:
the glob and the search keyword. 

    For instance, my pattern is:
    ```
    @*.html style
    ```
    It means: *At HTML file only, search for the word `style`*.

    Or another pattern:
    ```
    !*.html style
    ```
    Which means: *Exclude all HTML file, search for the word `style`*
    
    I use [Fish shell](https://fishshell.com/) as my shell script so I will
    write my function in Fish as below
    ```
    function vim_rg 
        set -l query $argv[1] # assign the argument to variable `query`

        # check if the query matches with the regex 
        set -l queries (string match -r '@(\S+)\s+(.+)' $query) 
        set -l ex_queries (string match -r '!(\S+)\s+(.+)' $query)

        # call rg command with corresponding glob
        if test -n "$queries" 
            rg --column --line-number --no-heading --color=always --smart-case --iglob $queries[2] -- $queries[3] || true
        else if test -n "$ex_queries"
            rg --column --line-number --no-heading --color=always --smart-case --iglob !$ex_queries[2] -- $ex_queries[3] || true
        else
            rg --column --line-number --no-heading --color=always --smart-case -- $query || true
        end
    end

    ```
    The above function will take the pattern as input, split into the glob
    and the real keyword, then run RipGrep command to do the search.
    Don't forget to make your function globally available for your system.
    For Fish, just put it in `~/.config/fish/functions/` directory.
    Now you can call the function from anywhere.
    
2. Update function `RipgrepFzf` in your vim config (or clone it into
another function) to use the above shell command as reload command
    ```
    function! RipgrepFzfWithGlob(query, fullscreen)
        let command_fmt = 'rg --column --line-number --no-heading --color=always --smart-case -- %s || true'
        let initial_command = printf(command_fmt, shellescape(a:query))
        let reload_command = "vim_rg {q}" "execute fish function vim_rg in fzf reload
        let spec = {'options': ['--phony', '--query', a:query, '--bind', 'change:reload:'.reload_command]}
        call fzf#vim#grep(initial_command, 1, fzf#vim#with_preview(spec), a:fullscreen)
    endfunction

    command! -nargs=* -bang RGGlob call RipgrepFzfWithGlob(<q-args>, <bang>0)

    ```
    Also create a Vim custom command `RGGlob` to call that function.
    
    As you can see from the above script, the `reload_command` uses `vim_rg` function.
    When user types in fzf search window, `vim_rg` will be called with one argument, the query string denoted by `{q}`.
    
3. Now just create some keymapping in Vim config to call the command `RGGlob`
    ```
    nnoremap <leader>fn :RGGlob !*Test*<space><CR>
    nnoremap <leader>ft :RGGlob @*Test*<space><CR>
    ```
    
    The first key mapping for search in non-test files, the second do the same in test files.


