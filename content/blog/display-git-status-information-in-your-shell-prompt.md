Title: Display git status information in your shell prompt!
Date: 2020-08-01
Category: Programming
Tags: Bash, Git
Summary: I have been using git frequently these days. Recently, I have been on the hunt for a script to display git status information in my bash prompt. This way I can easily see important git repository information in my shell prompt without having to type git status all of the time which would save frequent git users like myself a lot of time. However, many of the existing solutions that I found were either too simplistic or complete overkill for the task at hand, so I decided to create my own.

I have been using git frequently these days. Recently, I have been on the hunt for a script to display git status information in my bash prompt. This way I can easily see important git repository information in my shell prompt without having to type git status all of the time which would save frequent git users like myself a lot of time. However, many of the existing solutions that I found were either too simplistic or complete overkill for the task at hand, so I decided to create my own.

My implementation prints the name of the current git branch along with some other indicators:

* `'*'`: the repository is dirty
* `'?'`: untracked files exist in the repository
* `'='`: the local branch is in sync with the remote
* `'<'`: the local branch is behind of the remote
* `'>'`: the local branch is ahead of the remote
* `'<>'`: the local branch has diverged from the remote

The internals of the script take advantage of the git porcelain command and some relatively simple awk to parse its output. I felt that awk was the right tool for the job since awk made it easy to combine the grep and sed operations that I needed to use into one command. I also avoided the use of bash specific shell scripting features in effort to keep the script POSIX compliant, so given that your system has a POSIX compliant shell and awk, you should be able to use the script without any problems.

``` {.bash}
git_status() {
    STATUS="$(git status --branch --porcelain=v2 2>/dev/null)"
    if [ -z "$STATUS" ]; then
        return
    fi
    BRANCH="$(echo "$STATUS" | awk '$2 == "branch.head" { print $3 }')"
    DIRTY="$(echo "$STATUS" | awk '$1 ~ /[^#?]+/ { print $0 }')"
    if [ ! -z "$DIRTY" ]; then
        DIRTY_IND='*'
    else
        DIRTY_IND=''
    fi
    UNTRACKED="$(echo "$STATUS" | awk '$1 ~ /\?+/ { print $0 }')"
    if [ ! -z "$UNTRACKED" ]; then
        UNTRACKED_IND='?'
    else
        UNTRACKED_IND=''
    fi
    REMOTE="$(echo "$STATUS" | awk '$2 == "branch.ab" { gsub(/[+-]/, ""); print $3,$4 }')"
    case "$REMOTE" in
        '')
            REMOTE_IND='' ;;
        '0 0')
            REMOTE_IND='=' ;;
        '0 '*)
            REMOTE_IND='<' ;;
        *' 0')
            REMOTE_IND='>' ;;
        *)
            REMOTE_IND='<>' ;;
    esac
    INDICATORS="${UNTRACKED_IND}${DIRTY_IND}${REMOTE_IND}"
    if [ ! -z "$BRANCH" ] && [ ! -z "$INDICATORS" ]; then
        echo -n " (${BRANCH} ${INDICATORS})"
    elif [ ! -z "$BRANCH" ] && [ -z "$INDICATORS" ]; then
        echo -n " (${BRANCH})"
    elif [ -z "$BRANCH" ] && [ ! -z "$INDICATORS" ]; then
        echo -n " (${INDICATORS})"
    fi
}
```

Here is a snippet of my .bashrc file. I like to keep things tidy by checking to see if git is installed before using the PS1 loaded with the git status information.

``` {.bash}
export GIT_PS1="\[\033[1;32m\]\u@\h \[\033[1;33m\]\w\[\033[1;35m\]\$(git_status)\[\033[0m\]\n\$ "
export DEFAULT_PS1='\[\033[1;32m\]\u@\h \[\033[1;33m\]\w\[\033[0m\]\n\$ '

if [ -x "$(command -v git)" ]; then
    export PS1=$GIT_PS1
else
    export PS1=$DEFAULT_PS1
fi
```

Cheers! I hope you enjoy the script.
