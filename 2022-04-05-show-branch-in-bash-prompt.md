# Show current branch in bash prompt

```bash
# Git branch in terminal
parse_git_branch () {
  branch=$(git branch 2> /dev/null | sed -e '/^[^*]/d' -e 's/* \(.*\)/\1/')
  if [ ! -z "$branch" ]; then
    echo " ($branch)"
  fi
}

PS1='\[\033[01;32m\]\u\[\033[01;33m\]@\[\033[0m\]\[\033[01;32m\]\h\[\033[0m\] \[\033[01;34m\]\W\[\033[00m\]$(parse_git_branch) \$ '
PS2='\[\033[01;36m\]>'
```
