# Introduction
Misc setup for my dev environment

# Prerequsites

- Install [iTerm2](https://iterm2.com/)

- Setup ohmyzsh by following the [guide](./ohmyzsh-setup)

- Install [homebrew](https://brew.sh/)
```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

- Install utils
```
 brew install coreutils
 brew install vim
 brew install tmux
```

- Install [vim plugin manager](https://github.com/junegunn/vim-plug)
```
curl -fLo ~/.vim/autoload/plug.vim --create-dirs \
    https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim
```

- Install [tmux plugin manager](https://github.com/tmux-plugins/tpm)
```
 git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
```

- Copy <file> (vimrc, tmux.conf, zshrc) under $HOME/.<file>

- Install plugins

Open `vim` and execute `:PlugInstall` to install the plugins.
Open `tmux` and press `prefix` + `I` to install the plugins.

