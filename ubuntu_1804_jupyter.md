# Ubuntu 18.04にJupyter NotebookとIRubyをインストール

```
Ubuntu 18.04
anyenv
pyenv
  Python 3.7.7
  jupyter 1.0.0
rbenv
  Ruby 2.7.1
  iruby 0.4.0
```

Docker を使っていますが、まっさらな状態に戻してやり直したりしたいためです。





# Docker の用意

Dockerfile

```sh
FROM ubuntu:18.04

RUN apt-get update \
  && apt-get install -y sudo git wget build-essential nano

RUN useradd --create-home --gid sudo --shell /bin/bash user1 \
  && echo 'user1:pass' | chpasswd \
  && echo 'Defaults visiblepw'           >> /etc/sudoers \
  && echo 'user1 ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

USER user1

WORKDIR /home/user1

CMD ["/bin/bash"]
```

参考: [Dockerコンテナ内にsudoユーザを追加する - Qiita](https://qiita.com/iganari/items/1d590e358a029a1776d6)

イメージをビルドしてコンテナを起動。

```sh
docker build -t ubuntu_jupyter:trial .
docker run --rm -it -p8888:8888 ubuntu_jupyter:trial bash
```

以下はコンテナ内で作業しています。

----

PS1 が見にくいので変更。必須ではない。

```sh
echo "export PS1='----------------'\"\n\${PS1}\"" >> ~/.bashrc
exec bash -l
```





# anyenv, rbenv, pyenv のインストール

```sh
git clone https://github.com/anyenv/anyenv ~/.anyenv

export PATH="$HOME/.anyenv/bin:$PATH"
echo 'export PATH="$HOME/.anyenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(anyenv init -)"'               >> ~/.bashrc

exec bash -l

## メッセージにしたがって実行
yes | anyenv install --init

anyenv install rbenv
anyenv install pyenv
exec bash -l
```






# Ruby 2.7.1 のインストール

```sh
sudo apt install -y libssl-dev zlib1g-dev
rbenv install 2.7.1
```

<!--
real    2m48.010s
user    6m24.533s
sys     0m43.502s
-->

Docker のポートマッピングの確認。
先に疎通確認しておきます。

ホスト側から http://localhost:8888/ にアクセスできることを確認して
Ctrl-C で止める。

```sh
RBENV_VERSION=2.7.1 ruby -run -e httpd -- --port=8888 --bind-address=0.0.0.0 .
```






# Python 3.7.7 のインストール

```sh
# あとで必要になるので
sudo apt install -y libffi-dev libsqlite3-dev

env PYTHON_CONFIGURE_OPTS='--enable-shared' pyenv install 3.7.7
```

`env PYTHON_CONFIGURE_OPTS='--enable-shared'` は後で PyCall を使うための指定
https://github.com/mrkn/pycall.rb#note-for-pyenv-users






# Jupyter Notebook のインストール

```sh
# ModuleNotFoundError: No module named '_ctypes'
# ディレクトリと rbenv, pyenv の用意

mkdir jupyter
cd jupyter/
pwd #=> /home/user1/jupyter

rbenv local 2.7.1
pyenv local 3.7.7

ruby -v   #=> ruby 2.7.1p83 (2020-03-31 revision a0c7c23c9c) [x86_64-linux]
python -V #=> Python 3.7.7
```



```sh
# bundle init みたいなもの
# "venv.d" は任意のディレクトリ名
python -m venv venv.d

# 常に bundle exec してるみたいなモードになる
# モードを抜けたい場合は deactivate を実行する
. venv.d/bin/activate

# Bundler 環境での gem install みたいなもの
# Gemfile に追加して bundle install みたいなもの
pip install jupyter

# dockerでjupyter notebookが動く環境を付け加える作業 - Qiita
# https://qiita.com/tand826/items/0c478bf63ead75427782
# --ip=0.0.0.0 を指定しているのは Docker コンテナ内で実行してホスト側から参照するため
jupyter notebook --no-browser --ip=0.0.0.0

# ... 起動時にログイン用のトークンが表示される

# ホスト側で開く
# http://localhost:8888/
```

確認できたら Ctrl-C で止める。






# IRuby のインストール

```sh
# https://github.com/SciRuby/iruby
# のインストールの記述に従ってライブラリをインストール

sudo apt install -y libtool libffi-dev ruby ruby-dev make
sudo apt install -y libzmq3-dev libczmq-dev

bundle init

echo 'gem "ffi-rzmq"' >> Gemfile
echo 'gem "iruby"'    >> Gemfile
echo 'gem "pycall"'   >> Gemfile

bundle install --path=vendor/bundle

# こういうメッセージが出る。 pry とかあると便利？
Consider installing the optional dependencies to get additional functionality:
  * pry
  * pry-doc
  * awesome_print
  * gnuplot
  * rubyvis
  * nyaplot
  * cztop
  * rbczmq

bundle exec iruby register --force
## ~/.local/share/jupyter/kernels/ruby/kernel.json
## が生成される

cat ~/.local/share/jupyter/kernels/ruby/kernel.json
#=> {"argv":["/home/user/jupyter/vendor/bundle/ruby/2.7.0/bin/iruby","kernel","{connection_file}"],"display_name":"Ruby 2.7.1","language":"ruby"}

## この状態で jupyter 起動
jupyter notebook --no-browser --ip=0.0.0.0
## → Ruby 2.7.1 のノートブックが新規作成できるようになる

## なるが、次のようなエラーが jupyter を起動したターミナルに出る
## うまく動いていないっぽい

[I 07:06:44.327 NotebookApp] KernelRestarter: restarting kernel (3/5), new random ports
Traceback (most recent call last):
        2: from /home/user/jupyter/vendor/bundle/ruby/2.7.0/bin/iruby:23:in `<main>'
        1: from /home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/rubygems.rb:294:in `activate_bin_path'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/rubygems.rb:275:in `find_spec_for_exe': can't find gem iruby (>= 0.a) with executable iruby (Gem::GemNotFoundException)
```

これはおそらく rbenv + bundler のせいなので、
iruby コマンドのラッパー `iruby.sh` を用意して対処してみる。

他に良い方法があるかもしればいが、ここではひとまずこれで動きました。

```sh
cat <<'EOB' > iruby.sh
#!/bin/bash

JUPYTER_DIR=~/jupyter

export PYENV_ROOT="${HOME}/.anyenv/envs/pyenv"
export LIBPYTHON=${PYENV_ROOT}/versions/3.7.7/lib/libpython3.7m.so.1.0
export PYTHON=${JUPYTER_DIR}/venv.d/bin/python
# これでもいい？
# export PYTHON=${PYENV_ROOT}/shims/python

export RBENV_ROOT="${HOME}/.anyenv/envs/rbenv"
export PATH="${RBENV_ROOT}/bin:${PATH}"
eval "$(rbenv init -)"

rbenv shell 2.7.1

BUNDLE_GEMFILE=${JUPYTER_DIR}/Gemfile \
  bundle exec iruby "$@"
EOB
```

```sh
## 実行権限付ける
chmod u+x iruby.sh

## ~/.local/share/jupyter/kernels/ruby/kernel.json
## の iruby のパスを修正
nano ~/.local/share/jupyter/kernels/ruby/kernel.json

{
  "argv":[
    "/home/user1/jupyter/iruby.sh",  ...ここだけ修正
    "kernel",
    "{connection_file}"
  ],
  "display_name":"Ruby 2.7.1",
  "language":"ruby"
}

## もう一度 jupyter を起動
jupyter notebook --no-browser --ip=0.0.0.0

## ruby カーネルに接続できて動くようになった
```
