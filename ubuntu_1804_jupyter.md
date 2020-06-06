# Ubuntu 18.04にJupyter NotebookとIRubyをインストール

```
Ubuntu 18.04
anyenv
pyenv
  Python ...
rbenv
  Ruby 2.7.1
jupyter ...
iruby ...
```


Docker を使っていますが、まっさらな状態に戻してやり直したりしたいためです。

---

Dockerfile

```sh
FROM ubuntu:18.04

RUN apt-get update \
  && apt-get install -y sudo git wget build-essential

RUN useradd --create-home --gid sudo --shell /bin/bash user1 \
  && echo 'user1:pass' | chpasswd \
  && echo 'Defaults visiblepw'          >> /etc/sudoers \
  && echo 'user1 ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers

USER user1

WORKDIR /home/user1

# Dockerコンテナ内にsudoユーザを追加する - Qiita
# https://qiita.com/iganari/items/1d590e358a029a1776d6

CMD ["/bin/bash"]
```

イメージをビルド

```
docker build -t ubuntu_jupyter:trial .
```

コンテナを起動。

```
docker run --rm -it -p8888:8888 ubuntu_jupyter:trial bash
```

以下はコンテナ内で作業しています。

----

PS1 が見にくいので変更。必須ではない。

```sh
# .bashrc に追記
export PS1='----------------'"\n${PS1}"
```

anyenv, rbenv, pyenv のインストール

```sh
git clone https://github.com/anyenv/anyenv ~/.anyenv

export PATH="$HOME/.anyenv/bin:$PATH"
echo 'export PATH="$HOME/.anyenv/bin:$PATH"' >> ~/.bashrc

## なくてもよい？
## ~/.anyenv/bin/anyenv init

## メッセージにしたがって実行
echo 'eval "$(anyenv init -)"' >> ~/.bashrc

exec bash -l

## メッセージにしたがって実行
yes | anyenv install --init

anyenv install rbenv
anyenv install pyenv
exec bash -l
```

----
Ruby 2.7.1 のインストール

```
# /tmp/ruby-build.20200602115141.313.114NV4/ruby-2.7.1/lib/rubygems/core_ext/kernel_require.rb:92:in `require': cannot load such file -- openssl (LoadError)
sudo apt install -y libssl-dev

# The Ruby zlib extension was not compiled.
# ERROR: Ruby install aborted due to missing extensions
# Try running `apt-get install -y zlib1g-dev` to fetch missing dependencies.
sudo apt install -y zlib1g-dev

rbenv install 2.7.1
```

先に疎通確認しておきます。

```
## 確認
RBENV_VERSION=2.7.1 ruby -run -e httpd -- --port=8888 --bind-address=0.0.0.0 .
```

ホスト側から http://localhost:8888/ にアクセスできることを確認して
Ctrl-C で止める。

----

Python 3.7.7 のインストール

```sh
# あとで必要になるので
# ModuleNotFoundError: No module named '_ctypes'
# ModuleNotFoundError: No module named '_sqlite3'
sudo apt install -y libffi-dev libsqlite3-dev

env PYTHON_CONFIGURE_OPTS='--enable-shared' pyenv install 3.7.7
```

`env PYTHON_CONFIGURE_OPTS='--enable-shared'` は後で PyCall を使うための指定
https://github.com/mrkn/pycall.rb#note-for-pyenv-users

```
# --------------------------------
# ディレクトリの用意

mkdir jupyter
cd jupyter/
pwd #=> /home/user/jupyter

rbenv local 2.7.1
pyenv local 3.7.7

ruby -v #=> ruby 2.7.1p83 (2020-03-31 revision a0c7c23c9c) [x86_64-linux]
python -V #=> Python 3.7.7

# --------------------------------
# jupyter のインストール

# Bundler みたいなもの
# "venv.d" は何でもよい
python -m venv venv.d

# 常に bundle exec してるみたいなモードになる
# binstub みたいなもの？
# モードを抜けたい場合は deactivate を実行する
. venv.d/bin/activate

# Bundler 環境での gem install みたいなもの
pip install jupyter

# dockerでjupyter notebookが動く環境を付け加える作業 - Qiita
# https://qiita.com/tand826/items/0c478bf63ead75427782
jupyter notebook --no-browser --ip=0.0.0.0 --allow-root

# ... --allow-root は不要っぽい
# ... 起動時にログイン用のトークンが表示される

# ホスト側で開く
# http://localhost:8888/

# --------------------------------
# IRuby のインストール

# https://github.com/SciRuby/iruby
# のインストールの記述に従ってライブラリをインストール

sudo apt install -y libtool libffi-dev ruby ruby-dev make
sudo apt install -y libzmq3-dev libczmq-dev

bundle init

echo 'gem "ffi-rzmq"' >> Gemfile
echo 'gem "iruby"' >> Gemfile

bundle -v #=> Bundler version 2.1.4

bundle config set path 'vendor/bundle'
bundle install

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
# ~/.local/share/jupyter/kernels/ruby/kernel.json
# が生成される

cat ~/.local/share/jupyter/kernels/ruby/kernel.json
#=> {"argv":["/home/user/jupyter/vendor/bundle/ruby/2.7.0/bin/iruby","kernel","{connection_file}"],"display_name":"Ruby 2.7.1","language":"ruby"}

この状態で jupyter 起動
→ Ruby 2.7.1 のノートブックが新規作成できるようになる

なるが、次のようなエラーが jupyter を起動したターミナルに出る
うまく動いていないっぽい

[I 07:06:44.327 NotebookApp] KernelRestarter: restarting kernel (3/5), new random ports
Traceback (most recent call last):
        2: from /home/user/jupyter/vendor/bundle/ruby/2.7.0/bin/iruby:23:in `<main>'
        1: from /home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/rubygems.rb:294:in `activate_bin_path'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/rubygems.rb:275:in `find_spec_for_exe': can't find gem iruby (>= 0.a) with executable iruby (Gem::GemNotFoundException)

これはおそらく rbenv + bundler のせいなので、
iruby コマンドのラッパーを用意して対処してみる
他に良い方法があるかもだが、ここではひとまずこれで動いた。

-- find ~/.anyenv/envs/pyenv/ | grep libpython
-- /home/user/.anyenv/envs/pyenv/versions/3.7.7/lib/libpython3.7m.so
-- /home/user/.anyenv/envs/pyenv/versions/3.7.7/lib/libpython3.so
-- /home/user/.anyenv/envs/pyenv/versions/3.7.7/lib/python3.7/config-3.7m-x86_64-linux-gnu/libpython3.7m.a
-- /home/user/.anyenv/envs/pyenv/versions/3.7.7/lib/libpython3.7m.so.1.0

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

chmod u+x iruby.sh

# ~/.local/share/jupyter/kernels/ruby/kernel.json
# の iruby のパスを修正

sudo apt install -y nano
nano ~/.local/share/jupyter/kernels/ruby/kernel.json

↓こうなっている
{
  "argv":[
    "/home/user/jupyter/iruby.sh",
    "kernel",
    "{connection_file}"
  ],
  "display_name":"Ruby 2.7.1",
  "language":"ruby"
}

もう一度 jupyter を起動

----
F, [2020-06-03T07:21:27.905941 #14539] FATAL -- : Kernel died: uninitialized constant IRuby::SessionAdapter::PyzmqAdapter::PyCall
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/session_adapter/pyzmq_adapter.rb:7:in `rescue in load_requirements'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/session_adapter/pyzmq_adapter.rb:4:in `load_requirements'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/session_adapter.rb:7:in `available?'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/session_adapter.rb:63:in `block in select_adapter_class'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/session_adapter.rb:62:in `each_value'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/session_adapter.rb:62:in `select_adapter_class'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/session.rb:112:in `create_session_adapter'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/session.rb:12:in `initialize'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/kernel.rb:17:in `new'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/kernel.rb:17:in `initialize'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/command.rb:110:in `new'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/command.rb:110:in `run_kernel'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/command.rb:40:in `run'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/bin/iruby:5:in `<top (required)>'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/bin/iruby:23:in `load'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/bin/iruby:23:in `<top (required)>'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/bundler/cli/exec.rb:63:in `load'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/bundler/cli/exec.rb:63:in `kernel_load'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/bundler/cli/exec.rb:28:in `run'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/bundler/cli.rb:476:in `exec'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/bundler/vendor/thor/lib/thor/command.rb:27:in `run'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/bundler/vendor/thor/lib/thor/invocation.rb:127:in `invoke_command'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/bundler/vendor/thor/lib/thor.rb:399:in `dispatch'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/bundler/cli.rb:30:in `dispatch'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/bundler/vendor/thor/lib/thor/base.rb:476:in `start'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/bundler/cli.rb:24:in `start'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/gems/2.7.0/gems/bundler-2.1.4/libexec/bundle:46:in `block in <top (required)>'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/bundler/friendly_errors.rb:123:in `with_friendly_errors'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/gems/2.7.0/gems/bundler-2.1.4/libexec/bundle:34:in `<top (required)>'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/bin/bundle:23:in `load'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/bin/bundle:23:in `<main>'
bundler: failed to load command: iruby (/home/user/jupyter/vendor/bundle/ruby/2.7.0/bin/iruby)
NameError: uninitialized constant IRuby::SessionAdapter::PyzmqAdapter::PyCall
  /home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/session_adapter/pyzmq_adapter.rb:7:in `rescue in load_requirements'
  /home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/session_adapter/pyzmq_adapter.rb:4:in `load_requirements'
  /home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/session_adapter.rb:7:in `available?'
  /home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/session_adapter.rb:63:in `block in select_adapter_class'
  /home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/session_adapter.rb:62:in `each_value'
  /home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/session_adapter.rb:62:in `select_adapter_class'
  /home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/session.rb:112:in `create_session_adapter'
  /home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/session.rb:12:in `initialize'
  /home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/kernel.rb:17:in `new'
  /home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/kernel.rb:17:in `initialize'
  /home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/command.rb:110:in `new'
  /home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/command.rb:110:in `run_kernel'
  /home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/command.rb:40:in `run'
  /home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/bin/iruby:5:in `<top (required)>'
  /home/user/jupyter/vendor/bundle/ruby/2.7.0/bin/iruby:23:in `load'
  /home/user/jupyter/vendor/bundle/ruby/2.7.0/bin/iruby:23:in `<top (required)>'
----

PyCall がないと言われた。

echo 'gem "pycall"' >> Gemfile
bundle install

jupyter 起動 ... の前に、 iruby console で試すのがよさそう

F, [2020-06-03T07:26:40.668803 #15224] FATAL -- : Kernel died: <zmq.sugar.socket.Socket object at 0x7f92ed2710c0> is not a symbol nor a string
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/session.rb:85:in `send'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/kernel.rb:74:in `send_status'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/kernel.rb:36:in `run'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/command.rb:110:in `run_kernel'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/command.rb:40:in `run'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/bin/iruby:5:in `<top (required)>'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/bin/iruby:23:in `load'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/bin/iruby:23:in `<top (required)>'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/bundler/cli/exec.rb:63:in `load'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/bundler/cli/exec.rb:63:in `kernel_load'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/bundler/cli/exec.rb:28:in `run'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/bundler/cli.rb:476:in `exec'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/bundler/vendor/thor/lib/thor/command.rb:27:in `run'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/bundler/vendor/thor/lib/thor/invocation.rb:127:in `invoke_command'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/bundler/vendor/thor/lib/thor.rb:399:in `dispatch'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/bundler/cli.rb:30:in `dispatch'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/bundler/vendor/thor/lib/thor/base.rb:476:in `start'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/bundler/cli.rb:24:in `start'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/gems/2.7.0/gems/bundler-2.1.4/libexec/bundle:46:in `block in <top (required)>'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/2.7.0/bundler/friendly_errors.rb:123:in `with_friendly_errors'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/lib/ruby/gems/2.7.0/gems/bundler-2.1.4/libexec/bundle:34:in `<top (required)>'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/bin/bundle:23:in `load'
/home/user/.anyenv/envs/rbenv/versions/2.7.1/bin/bundle:23:in `<main>'

/home/user/jupyter/iruby.sh notebook --no-browser --ip=0.0.0.0
# → これは jupyter notebook での起動と同じっぽい

/home/user/jupyter/iruby.sh console
/home/user/jupyter/iruby.sh console --kernel=ruby

----
F, [2020-06-03T07:31:53.009220 #16130] FATAL -- : Kernel heartbeat died: <class 'ModuleNotFoundError'>: No module named 'iruby'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/pycall-1.3.1/lib/pycall.rb:62:in `import_module'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/pycall-1.3.1/lib/pycall.rb:62:in `import_module'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/session_adapter/pyzmq_adapter.rb:29:in `heartbeat_loop'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/session.rb:53:in `block in setup_heartbeat'
W, [2020-06-03T07:31:53.009317 #16130]  WARN -- : parent process poller thread started.
F, [2020-06-03T07:31:53.050593 #16130] FATAL -- : Kernel died: <zmq.sugar.socket.Socket object at 0x7f3f77cdb0c0> is not a symbol nor a string
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/session.rb:85:in `send'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/kernel.rb:74:in `send_status'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/kernel.rb:36:in `run'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/command.rb:110:in `run_kernel'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/lib/iruby/command.rb:40:in `run'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/gems/iruby-0.4.0/bin/iruby:5:in `<top (required)>'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/bin/iruby:23:in `load'
/home/user/jupyter/vendor/bundle/ruby/2.7.0/bin/iruby:23:in `<top (required)>'
...
----


```



docker-compose run だとポートマッピングの指定が効かないので up で実行する
docker-compose -f docker-compose_rbenv.yaml up -d service_main
docker exec -it {cont} bash

RBENV_VERSION=2.7.1 ruby -run -e httpd -- --port=8888 --bind-address=0.0.0.0 .
RBENV_VERSION=2.7.1 ruby -run -e httpd -- --port=9999 --bind-address=0.0.0.0 .
