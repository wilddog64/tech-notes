# How to reconnect back a lost nvim session from tmux

Sometimes you may lost your nvim tmux sessions. Here are some ways to find them and connect back.

## List them and their sessions


```bash 

# list all tmux servers (sockets)
ls -al /tmp/tmux-$(id -u)

# list sessions for each server
for s in /tmp/tmux-$(id -u)/*; do
    echo "== $s =="; tmux -S "$s" list-sessions 2>/dev/null || true
done
```

If you see sessions under a non-default socket (like `/tmp/tmux-1000/default`), you can attach to it using:

```bash
tmux -S /tmp/tmux-1000/default attach
```

## Find which pane has Neovim

Search every tmux server for panes whose current command is `nvim`:

```bash
for s in /tmp/tmux-$(id -u)/*; do
    echo "== $s =="; 
    tmux -S "$s" list-panes -a -F '#{session_name}:#{window_index}.#{pane_index} #{pane_current_command}' | grep nnvim || true
done
```

this will prints lines like:

```bash
/tmp/tmux-1000/default work:2.1 12345 /dev/pts/3 nvim
```

Then attach and jump straight there:

```bash
tmux -S /tmp/tmux-1000/default attach -t work

# once inside that server:
tmux select-windows -t 2
tmux select-pane -t 1
```

## If you only have the PID from `ps`

Map the PID (or its TTY) to a tmux pane:

```bash
PID=12345  # replace with your nvim PID
tmux list-panes -a -F '#{session_name}:#{window_index}.#{pane_index} #{pane_pid} #{pane_tty}' 2>/dev/null | ak -v p=$PID '$2 == p'

# or if you have the TTY
TTY=/dev/pts/3  # replace with your nvim TTY
tmux list-panes -a -F '#{session_name}:#{window_index}.#{pane_index} #{pane_pid} #{pane_tty}' 2>/dev/null | ak -v t=$TTY '$3 == t'
```

Oncce have the target like `work:2.1`, attach/select as above.

## Common gotchas

* `fg` only works for job in the `same shell's job table`. If you closed that shell or the job was in another terminal, `fg` can't help
* If your `nvim` wasn't started inside tmux, you can't __reattach__ to its TTY. In that case, save/recover:
  * Try `nvim -r` to restore from swap
  * Or gracefully `kill -TERM <pid>` then `nvim -r <file>`
