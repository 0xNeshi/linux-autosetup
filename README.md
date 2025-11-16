1. Create an executable bash file (_*.sh_) like:

```bash
touch setup.sh
chmod +x ./setup.sh
```

2. Copy the contents below into the file you've just created (make sure to update variables `EMAIL` and `NAME` at the top of the file):

```bash
#!/usr/bin/env bash

sep() {
  echo
  echo "------------------"
  echo
}

# IMPORTANT!!!
# REPLACE THESE WITH YOUR VALUES!!!
EMAIL="email"
NAME="name"

########### install packages
sudo apt update && sudo apt upgrade -y && sudo apt install curl wget build-essential -y && sep

# turn power management off for wifi
sudo sed -i 's/3/2/' /etc/NetworkManager/conf.d/default-wifi-powersave-on.conf
sep

cd ~/Downloads && sep

if [[ $* == *--chrome* ]]; then
  # Google Chrome
  echo "Install Google Chrome..."
  wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
  sudo dpkg -i google-chrome-stable_current_amd64.deb
  # add to favorites
  if [ "$1" = "ubuntu" ]; then
    gsettings set org.gnome.shell favorite-apps "$(gsettings get org.gnome.shell favorite-apps | sed s/.$//), 'google-chrome.desktop']"
  fi
  sudo apt --fix-broken install
  google-chrome --version
  sep
fi

# KeePassXC
echo "Install KeePassXC..."
sudo add-apt-repository ppa:phoerious/keepassxc -y
sudo apt update
sudo apt install keepassxc -y
sudo apt --fix-broken install
keepassxc -v
sep

# git
echo "Install git..."
sudo apt install git-all -y
git config --global user.name "$NAME"
git config --global user.email "$EMAIL"
git config --global core.editor "code --wait"
git --version
sep

# set git branch in terminal
cat <<'EOF' >>~/.bashrc
parse_git_branch() {
    git branch 2>/dev/null | sed -e '/^[^*]/d' -e 's/* \(.*\)/(\1)/'
}
export PS1="\[\e[32m\]\w \[\e[91m\]\$(parse_git_branch)\[\e[00m\]$ "
EOF
sep

# VS Code
echo "Install VS Code..."
sudo apt-get install gpg
wget -qO- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor >packages.microsoft.gpg
sudo install -D -o root -g root -m 644 packages.microsoft.gpg /etc/apt/keyrings/packages.microsoft.gpg
sudo sh -c 'echo "deb [arch=amd64,arm64,armhf signed-by=/etc/apt/keyrings/packages.microsoft.gpg] https://packages.microsoft.com/repos/code stable main" > /etc/apt/sources.list.d/vscode.list'
sudo apt install apt-transport-https
sudo apt update
sudo apt install code
code -v
sep

# nvm (Node Version Manager)
echo "Install nvm (Node Version Manager)..."
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash
\. "$HOME/.nvm/nvm.sh"
nvm install 22
node -v
nvm current
npm -v
sep

# install Go language
echo "Install Go..."
wget https://go.dev/dl/go1.24.6.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.24.6.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >>$HOME/.profile
source $HOME/.profile
go version
sep

# install Rust
# Toolchain 1.81.0 is necessary to work with Stylus
echo "Install Rust..."
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y
source $HOME/.profile
source $HOME/.bashrc
rustc --version
cargo install cargo-expand
sep

# install Cairo
echo "Install asdf..."
wget https://github.com/asdf-vm/asdf/releases/download/v0.18.0/asdf-v0.18.0-linux-amd64.tar.gz
mkdir $HOME/.asdf
echo 'export PATH="$PATH:/$HOME/.asdf"' >>$HOME/.bashrc
sudo tar -C $HOME/.asdf -xzf asdf-v0.18.0-linux-amd64.tar.gz
echo 'export PATH="${ASDF_DATA_DIR:-$HOME/.asdf}/shims:$PATH"' >>$HOME/.bashrc
echo '. <(asdf completion bash)' >>$HOME/.bashrc
source $HOME/.bashrc

echo "Install Scarb..."
asdf plugin add scarb
asdf install scarb latest
asdf set -u scarb latest
echo "Install Starkli..."
curl https://get.starkli.sh | sh
source ~/.bashrc
starkliup
sep

# install Starknet Foundry
echo "Install Starknet Foundry..."
curl -L https://raw.githubusercontent.com/foundry-rs/starknet-foundry/master/scripts/install.sh | sh
source $HOME/.bashrc
snfoundryup
sep

# yarn
echo "Install yarn..."
corepack enable
sep

# SSH key
echo "Create new SSH key..."
ssh-keygen -q -t ed25519 -C "$EMAIL" -f ~/.ssh/id_ed25519 <<<y >/dev/null 2>&1
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519
ssh-keygen -p -f ~/.ssh/id_ed25519
sep

# install GH CLI
echo "Install GitHub CLI..."
(type -p wget >/dev/null || (sudo apt update && sudo apt-get install wget -y)) &&
  sudo mkdir -p -m 755 /etc/apt/keyrings &&
  wget -qO- https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg >/dev/null &&
  sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg &&
  echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list >/dev/null &&
  sudo apt install gh -y

# upload SSH key to GH
echo "Authenticate to GitHub & Upload SSH key..."
gh auth login -w
gh config set git_protocol ssh --host github.com
gh ssh-key add ~/.ssh/id_ed25519.pub --type signing
gh ssh-key add ~/.ssh/id_ed25519.pub
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
sep

# Docker setup
echo "Installing Docker..."
# Add Docker's official GPG key:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update

sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# uncomment the below line to create 'docker' group if it does not exist
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
sep

# create /repos folder
echo "Create /repos folder..."
cd ~
mkdir repos
cd repos
sep

# clone OZ Stylus library
echo "Clone OpenZeppelin Stylus repo..."
git clone git@github.com:OpenZeppelin/rust-contracts-stylus.git
sep

# install nextest crate
echo "Installing Stylus cargo dependencies..."
cargo install cargo-nextest --locked
sep

# install Foundry RS
curl -L https://foundry.paradigm.xyz | bash
source $HOME/.bashrc
foundryup

# set up Arbitrum Stylus
echo "Setup Arbitrum Stylus tools..."
sudo apt-get install pkg-config libssl-dev -y
cargo install cargo-stylus@0.6.1
sep

# install Solidity (version for Stylus)
echo "Install Solidity compiler..."
sudo add-apt-repository ppa:ethereum/ethereum
sudo apt-get update
sudo apt-get install solc
sep

# clone Exercism Cairo
echo "Clone Exercism Cairo repo..."
git clone git@github.com:exercism/cairo.git exercism-cairo
sep

# remove redundant packages
echo "Cleaning up..."
sudo apt autoremove -y
sep

# add aliases
cat <<EOT >>~/.bash_aliases
# bun aliases
alias bi='bun install'

# custom git aliases
alias ga='git add'
alias gaa='git add .'
alias gaac='git add . && git commit -m'
alias gaaca='function gaacaf(){ git add .; git commit --amend --no-edit; }; gaacaf'
alias gaacam='function gaacaf(){ git add .; git commit --amend; }; gaacaf'
alias gaacap='function gaacapf(){ git add .; git commit --amend --no-edit; git push --force-with-lease; }; gaacapf'
alias gaacapm='function gaacapf(){ git add .; git commit --amend; git push --force-with-lease; }; gaacapf'
alias gaacp='function gaacpf(){ git add .; git commit -m "\$1"; git push --force-with-lease; }; gaacpf'
alias gaacpf='function gaacpff(){ git add .; git commit -m "\$1"; git push --force; }; gaacpff'
alias gac='function gacf(){ git add "\$1"; git commit -m "\$2"; }; gacf'
alias gaca='function gacaf(){ git add "\$1"; git commit --amend --no-edit; }; gacaf'
alias gacam='function gacamf(){ git add "\$1"; git commit --amend; }; gacamf'
alias gacap='function gacapf(){ git add "\$1"; git commit --amend --no-edit; git push --force-with-lease; }; gacapf'
alias gacapm='function gacapmf(){ git add "\$1"; git commit --amend; git push --force-with-lease; }; gacapmf'
alias gacp='function gacpf(){ git add "\$1"; git commit -m "\$2"; git push --force-with-lease; }; gacpf'
alias gacpf='function gacpff(){ git add "\$1"; git commit -m "\$2"; git push --force; }; gacpff'
alias gb='git branch'
alias gbdmr='git branch -D master'
alias gc='git commit -m'
alias gca='git commit --amend --no-edit'
alias gcam='git commit --amend -m'
alias gcb='git checkout -b'
alias gco='git checkout --ours .'
alias gcbp='function gcbpf(){ git checkout -b "\$1"; git push --set-upstream origin "\$1"; }; gcbpf'
alias gd='git diff'
alias gf='git fetch'
alias gfopr='function gfoprf(){ git fetch origin pull/"$1"/head:"$2" && git switch "$2"; }; gfoprf'
alias gfs='git fetch && git switch'
alias gl='git log --oneline'
alias gmom='git merge origin/main'
alias gmomr='git merge origin/master'
alias gp='git push --force-with-lease'
alias gpf='git push --force'
alias gpl='git pull'
alias gpu='git push --set-upstream origin'
alias gr='git rebase'
alias gra='git rebase --abort'
alias grc='git add . && git rebase --continue'
alias grh='git reset --hard'
alias grm='git rebase main'
alias grmr='git rebase master'
alias grom='git rebase origin/main'
alias gromrp='git rebase origin/master && git push --force-with-lease'
alias gromr='git rebase origin/master'
alias gron='git rebase --onto'
alias grs='git rebase --skip'
alias gs='git switch'
alias gsm='git switch main'
alias gsmpl='git switch main && git pull'
alias gsmr='git switch master'
alias gsmrpl='git switch master && git pull'
alias gst='git stash'
alias gstp='git stash pop'

# Exercism aliases
alias exfc='~/repos/exercism-cairo/bin/fetch-configlet'
alias exc='~/repos/exercism-cairo/bin/configlet'
alias excc='exc create'
alias exccce='exc create --concept-exercise'
alias exccpe='exc create --practice-exercise'
alias exf='exc fmt'
alias exl='exc lint'
alias exuuid='exc uuid | tr -d "\n" | xclip -selection clipboard'

# custom yarn aliases
alias y='yarn'
alias yb='GENERATE_SOURCEMAP=false yarn run build'
alias ybgst='GENERATE_SOURCEMAP=true yarn run build'
alias yc='yarn compile'
alias ycc='yarn compile clean'
alias yd='yarn deploy'
alias ydnm='yarn deploy --network mumbai --yes'
alias ydnl='yarn deploy --network localhost --yes'
alias ys='yarn start'
alias yt='yarn run test'
alias ytg='yarn run test --grep'
alias ytnc='yarn run test --no-compile'
alias ytncg='yarn run test --no-compile --grep'
alias yta='yarn run test --watchAll'
alias yyb='yarn && GENERATE_SOURCEMAP=false yarn build'
alias yys='yarn && yarn start'
alias yyt='yarn && yarn test'
alias yyta='yarn && yarn test --watchAll'

# custom hardhat aliases
alias hh='npx hardhat'
alias hhc='npx hardhat compile'
alias hhnl='npx hardhat --network localhost'
alias hhn='npx hardhat node'

# scarb aliases
alias st='scarb test'
alias stii='scarb test -- --include-ignored'
alias sr='scarb run'
alias scr='scarb cairo-run'
alias sb='scarb build'
alias sbt='scarb build --test'

# OZ Stylus aliases
alias nitro='./scripts/nitro-devnode.sh'
alias e2e='./scripts/e2e-tests.sh'
alias cnr='cargo nextest run'
alias cnf='cargo +nightly fmt --all'
alias clippy='cargo clippy --all-targets --all-features -- -D warnings -D clippy::pedantic'
alias clippyf='cargo clippy --all-targets --all-features --fix -- -D warnings -D clippy::pedantic'

# open .bashrc alias
alias cbash='code ~/.bashrc'
alias cbasha='code ~/.bash_aliases'
alias cprof='code ~/.profile'
alias sbash='source ~/.bashrc'
alias sprof='source ~/.profile'

# custom aliases
alias c='clear'
alias cd1='cd ..'
alias cd2='cd ../..'
alias cd3='cd ../../..'
alias cd4='cd ../../../..'
alias cd5='cd ../../../../..'
alias e='exit'
alias suup='sudo apt update && sudo apt upgrade -y && asdf install scarb latest && sudo apt autoremove -y'
EOT

# install Serbian language pack
sudo apt -y install $(check-language-support -l sr)

########### TODO
echo "----------------"
echo "TODO"
echo "----------------"
echo "Setup KeePassXC, see https://keepassxc.org/docs/KeePassXC_GettingStarted#_setup_browser_integration"
```

3. Run the script with `./setup.sh`.
