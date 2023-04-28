+++
title = "Fuzzy Search with fzf"
description = "How I use fzf for fuzzy searching everything"
date = 2023-04-28
slug = "fuzzy_search_fzf"
draft = false

[taxonomies]
tags = ["Workflows"]
+++

[fzf](https://github.com/junegunn/fzf) is a powerful tool for fuzzy searching, and can greatly enhance your workflow. Here are some ways that I use `fzf`:

### 1. Changing directories with ease

To quickly change directories, I use a bash alias that combines the speed of `fd` (an alternative to `find`) and the convenience of `fzf`.

```bash
cdh () {
        cd $(fd --type d . "/home/sarat" | fzf)
}
```

This command lists all directories under the /home/sarat directory (replace this with your desired directory) and pipes them to fzf, allowing you to select the desired directory with fuzzy search.

For example, if I need to search for the personal directory under `$HOME/projects/`, I just run cdh, search for personal, and hit enter. This command will change the current directory to `$HOME/projects/personal/`.

<script async id="asciicast-GT83gj3eMZMngUsyevFH5liLo" src="https://asciinema.org/a/GT83gj3eMZMngUsyevFH5liLo.js"></script>

### 2. Easily connecting to remote servers.

When you have many servers defined in your `.ssh/config` file, it can be difficult to find the one you need quickly. This is where `fzf` comes in handy. Here's a bash alias I use to search for and SSH into a server:

```bash
sshf () {
        hosts=$(grep "^Host " ~/.ssh/config | awk '{print $2}' | grep -v "*" | grep -v "*$")
        selected_host=$(echo "$hosts" | fzf --height=50% --reverse --prompt="SSH into: ")
        if [ -n "$selected_host" ]
        then
                ssh "$selected_host"
        fi
}
```

This command gets all of the hosts defined in your .ssh/config file, filters out any lines containing `*`, and pipes them to fzf. fzf then displays a searchable list of hosts, allowing you to easily select the one you want to SSH into. If you select a host, the command then uses ssh to connect to the selected host.

<script async id="asciicast-RMnNzfxXibAMtgQGhnBgNoKFu" src="https://asciinema.org/a/RMnNzfxXibAMtgQGhnBgNoKFu.js"></script>

### 3. ZSH Integration with Fzf

If you use ZSH as your shell, `fzf` can be integrated to help you search through your history more easily. I use `Oh My ZSH` to enhance my ZSH experience, and the `fzf` plugin makes it simple to search through my command history using fuzzy searching.

To integrate `fzf` with ZSH, you can add the following line to your `.zshrc` file:

```bash
plugins=(... fzf)
```

With this plugin enabled, you can search your command history using Ctrl + r. When you press this key combination, your history will be piped into fzf, allowing you to quickly and easily search through your past commands.

<script async id="asciicast-6ZKgW7ODpeNS7RrLHAGTam03o" src="https://asciinema.org/a/6ZKgW7ODpeNS7RrLHAGTam03o.js"></script>

Another useful plugin that makes use of `fzf` is [fzf-tab](https://github.com/Aloxaf/fzf-tab). This plugin replaces the default tab behavior in your shell and feeds the list of options to `fzf`, allowing you to fuzzy search and select the option you want.

For example, if you're trying to `cd` into a directory with a long and complicated name, instead of typing out the entire name or using tab to cycle through options, you can simply start typing the name and `fzf-tab` will present a list of matching directories. You can then select the directory you want and `cd` into it.

Using `fzf-tab` can save you time and reduce the frustration of tab cycling, making it another great way to use `fzf`.

## Conclusion

In this post, I've shared some of the workflows I use where `fzf` has improved my productivity and reduced my frustration. `fzf` is an incredibly versatile tool that can be used creatively to search and select anything using fuzzy searching.

What's great about `fzf` is that it is highly customizable. You can add options to preview a file, select multiple items, and more. I'm always finding new ways to incorporate `fzf` into my workflows to improve my productivity.

If you haven't tried `fzf` yet, I highly recommend giving it a try. It's a powerful tool that can help streamline your workflow and make your life easier on the command line.
