## Terminology

| Term   | Description                                                                                                            |
| ------ | ---------------------------------------------------------------------------------------------------------------------- |
| Buffer | A buffer contains the text of the file and is what you edit                                                            |
| Window | Contains a buffer to display. Windows can be closed but the buffer can remain in memory (see `:h buffer`)              |
| Tabs   | Is another viewport. You can have many windows/splits open per tab (see `:h tab`)                                      |
| Splits | A split refers to splitting the viewport in n sections of various size and orientation to display windows (`:h split`) |

## Motions

### Basic commands

`h, j, k, l, w, b`

`a, A, o, O`

`r, x, d, y, yy, dd`

### Advanced commands

`c, cc`

`_, 0, $, D, C, S, f, , , ;, t, F, T`

`ctrl d/u`

`di{`

## Patterns of Vim Motions

`cmd` + `count` + `motion`

where `cmd` is the command, such as `d` for delete, `count` is the repeat number of times, and `motion` is a move over the text, such as `w` for word.

### Vim Commands

| Key Combination               | Description                                                  |
| ----------------------------- | ------------------------------------------------------------ |
| `nvim` + `filename` + `enter` | Start `neovim` from the shell prompt                         |
| `:`                           | Enter command mode                                           |
| `esc` + `:wq` + `enter`       | Write changes and quit vim                                   |
| `ZZ`                          | quit vim and save file if there are detected changes         |
| `esc` + `:q!` + `enter`       | Quit without saving changes                                  |
| `h`                           | Move cursor left                                             |
| `j`                           | Move cursor down                                             |
| `k`                           | Move cursor up                                               |
| `l`                           | Move cursor right                                            |
| `i`                           | Switch to Insert mode                                        |
| `esc`                         | Return to normal mode or cancel an unwanted command          |
| `x`                           | Delete the current character under the cursor                |
| `r` + `newchar`               | Replace a character under the cursor with `newchar`          |
| `u`                           | Undo                                                         |
| `U`                           | Undo all the changes on a line                               |
| `ctrl` + `r`                  | Redo                                                         |
| `w`                           | Move by a word                                               |
| `b`                           | Move backward by a word                                      |
| `y`                           | Yank (or copy)                                               |
| `yy`                          | Yank the entire line. Works without highlighting             |
| `p`                           | Paste yanked or recently deleted text after the cursor       |
| `%`                           | Move between braces or parentheses                           |
| `a`                           | Move the cursor to the end of the word and enter insert mode |
| `A`                           | Move the cursor to the end of the line and enter insert mode |
| `V`                           | Select current line                                          |
| `:reg`                        | Command register                                             |

### Horizontal Movements

| Key Combination                        | Description                                                          |
| -------------------------------------- | -------------------------------------------------------------------- |
| `0`                                    | Move to the start of a line                                          |
| `_`                                    | Go to the beginning character                                        |
| `$`                                    | Go to the end character                                              |
| `dw`                                   | Delete from the cursor up to the next word                           |
| `d$`                                   | Delete everything to the end of the line                             |
| `dd`                                   | Delete a whole line                                                  |
| `f` + `char`                           | Move forward and on the specified character (`char`)                 |
| `t` + `char`                           | Move forward but **not on** the specified character                  |
| `,`                                    | Repeat the above command forward                                     |
| `;`                                    | Repeat the above command backward                                    |
| `F` + `char`                           | Go backward to the specified character                               |
| `T` + `char`                           | Go backward but **not on** the specified character                   |
| `yf` + `char`                          | Yank up to and include the specified character                       |
| `yt` + `char`                          | Yank up to the specified character                                   |
| `vF` + `char`                          | Highlight backward including the specified character                 |
| `vT` + `char`                          | Highlight backward but **not** including the specified character     |
| `I`                                    | Go to beginning of the line but in insert mode                       |
| `A`                                    | Go to the end of the line but in insert mode                         |
| `o`                                    | Make a new line below and go into insert mode                        |
| `O`                                    | Make a new line above and go into insert mode                        |
| `c`                                    | The change operator. Use similar to `d` but puts you in insert mode. |
| The format is `c` + `count` + `motion` |
| `cc`                                   | Equivalent of `dd` and puts you in insert mode                       |

### Vertical Movements

| Key Combination     | Description                                                           |
| ------------------- | --------------------------------------------------------------------- |
| `number` + `motion` | Repeat the `motion` the specified `number` of times. For example `5k` |
| `{`                 | Move up by one paragraph                                              |
| `}`                 | Move down by one paragraph                                            |
| `ctrl` + `u`        | Move up by half a page                                                |
| `ctrl` + `d`        | Move down by half a page                                              |
| `gg`                | Move to the top of the file                                           |
| `G`                 | Move down to the bottom of the file                                   |
| `ctrl` + `g`        |                                                                       |

### Search
| Key Combination     | Description                                                                                                                                 |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `/` + `search term` | Search for text. Press `n` to go to the next result and `N` to go back to the previous result                                               |
| `?` + `search term` | Reverse search (i.e. backward search within the file)                                                                                       |
| `*`                 | Copies the word the cursor is on and loads it into the search. Press `n` to go to the next result and `N` to go back to the previous result |
| `#`                 | Copies the word the cursor is on and loads into the reversed search. Press `n` to go to previous result and `N` to go to the next result    |
| `:%s/old/new`       | Find an replace                                                                                                                             |
| `:%s/old/new/g`     | Find and replace in the whole buffer                                                                                                        |

## Advanced Movements

### Horizontal Text Editing

| Key Combination | Description                                                    |
| --------------- | -------------------------------------------------------------- |
| `vi(`           | Select everything within parentheses `()`                      |
| `vi{`           | Select everything within braces `{}`                           |
| `va(`           | Select everything within and including parentheses `()`        |
| `va{`           | Select everything within and including braces `{}`             |
| `ya(`           | Yank everything within including the parentheses `()`          |
| `ya{`           | Yank everything within including the braces `{}`               |
| `viw`           | Select the word under the cursor                               |
| `viW`           | Select all contiguous characters between adjacent white spaces |
| `yiw`           | Yank the word under the cursor                                 |
| `yiW`           | Yank all contiguous characters between adjacent white spaces   |

### Vertical Text Editing

| Key Combination | Description                               |
| --------------- | ----------------------------------------- |
| `gg`            | Go to top of the file                     |
| `G`             | Go to the bottom of the file              |
| `:line number`  | Go to the line denoted by the line number |
| `num` + `j, k`  | Go up or down by `num` lines              |
| `{`             | Go up by a paragraph                      |
| `}`             | Go down by a paragraph                    |
| `ctrl` + `u`    | Go up by 1/2 a page                       |
| `ctrl` + `d`    | Go down by 1/2 a page                     |

### Combos

`va{Vd`

### Macros

[Macros: Vim Commands you NEED TO KNOW #3](https://www.youtube.com/shorts/vEToFNSuqrk)

[GitHub - iggredible/Learn-Vim: Learning Vim and Vimscript doesn't have to be hard. This is the guide that you're looking for 📖](https://github.com/iggredible/Learn-Vim/tree/master?tab=readme-ov-file)

[30 Vim commands you NEED TO KNOW (in just 10 minutes)](https://www.youtube.com/watch?v=RSlrxE21l_k)

### Find and replace

[Neovim – how to do project-wide find and replace?](https://www.youtube.com/watch?v=9JCsPsdeflY)

