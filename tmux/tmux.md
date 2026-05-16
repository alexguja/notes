## Basic configuration

```sh
# ~/.tmux.conf

set -g default-terminal "screen-256color"
set -g mouse on

set -g base-index 1
set -g pane-base-index 1
set-window-option -g pane-base-index 1
set-option -g renumber-windows on
set-option -g status-position top

set-window-option -g mode-keys vi

set -g @catppuccin_flavour 'mocha'

set -g prefix C-Space
unbind C-b
bind-key C-Space send-prefix

unbind r
bind r source-file ~/.tmux.conf

unbind %
bind | split-window -h -c "#{pane_current_path}"

unbind '"'
bind - split-window -v -c "#{pane_current_path}"

# tmux plugins
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'catppuccin/tmux'
set -g @plugin 'christoomey/vim-tmux-navigator'
set -g @plugin 'tmux-yank'

# Initialize TMUX plugin manager (keep this line at the very bottom of tmux.conf)
run '~/.tmux/plugins/tpm/tpm'
```

### Sessions

```sh
# Sessions
tmux new -s "SESSION_NAME"   # creates a new session
tmux new-session -d -s "SESSION_NAME" # create a named session in detach mode

tmux ls                      # list created sessions
tmux list-sessions           # list created sessions
tmux attach                  # attach to a session
tmux a                       # attach to a session
tmux attach-session "SESSION_NAME" # go into an existing session

tmux detach          # detach (exit) from current session
prefix + d           # detach (exit) from current session
tmux kill-server     # terminate all sessions

prefix + :source-file ~/.tmux.conf # load config for the 1st time
prefix + s              # pop up sessions list
prefix + w              # show a list of sessions and previews
prefix + :kill-session  # kill current session

prefix + ( # switch to the next tmux session
prefix + ) # switch to the prev tmux session
Ctrl + d   # end session

# Switch sessions based on name
tmux switch-client -t "SESSION_NAME" # target a session
tmux switch-client -t "WINDOW_NAME"  # target a window
tmux switchc -t "SESSION_NAME"

# Windows
tmux new-window  # new window for the same session
tmux new-window -n "WINDOW_NAME" # new named window
prefix + c       # new window for the same session
prefix + n       # next window
prefix + p       # prev window
prefix + number  # navigate to nth window
Ctrl + d         # end current window and jump back to prev window
								 # note it doesn't kill the session.
# Panes
exit                         # close a pane						 
							
# Misc	 
tmux new-session -d -s "SESSION_NAME" -n "WINDOW_NAME" # create new named session and window
tmux new-session -s "SESSION_NAME" -d -c "$HOME/path/to/project"
tmux new-window -n "WINDOW_NAME" -c "$HOME/path/to/project"
tmux new-window -n "WINDOW_NAME" [some terminal cmd] # specify a cmd like npm run dev
```

```sh
ln -s ~/dotfiles/nvim ~/.config/nvim
ln -s ~/dotfiles/.tmux.conf ~/.tmux.conf
```

https://www.youtube.com/watch?v=DzNmUNvnB04