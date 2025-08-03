## 安装kubens/kubectx

```properties
git clone https://github.com/ahmetb/kubectx /k0s/kubectx
cp /k0s/kubectx/kubectx /usr/local/bin/kubectx
cp /k0s/kubectx/kubens /usr/local/bin/kubens
```
## 别名生效

```properties
vim ~/.bashrc
# 添加---------------------
alias k='kubectl'
alias kn='kubens'
-----------------------------------
source ~/.bash_profile
```



## 安装kube-ps1

```properties
git clone https://github.com/marvinpan8/kube-ps1.git /root/kube-ps1
```

修改符号环境变量，增加别名，并生效所有命令

```properties
cat << EOF >> ~/.bash_profile
if [ -f /jrtz/kube-ps1/kube-ps1.sh ]; then
    KUBE_PS1_SYMBOL_DEFAULT=*
    PS1='[\u@\h \W $(kube_ps1)]\$ '
    source /jrtz/kube-ps1/kube-ps1.sh
fi
FOE
-----------------------------------
source ~/.bash_profile
```

常用命令

If you want to stop showing Kubernetes status on your prompt string temporarily run `kubeoff`. To disable the prompt for all shell sessions, run `kubeoff -g`. You can enable it again in the current shell by running `kubeon`, and globally with `kubeon -g`.

```properties
kubeon     : turn on kube-ps1 status for this shell.  Takes precedence over
             global setting for current session
kubeon -g  : turn on kube-ps1 status globally
kubeoff    : turn off kube-ps1 status for this shell. Takes precedence over
             global setting for current session
kubeoff -g : turn off kube-ps1 status globally
```

## 安装 Kube-prompt

Kube-prompt 使用 Go 语言开发，天生良好的跨平台性。安装起来非常简单，只需下载各平台对应的二进制版本就可以开箱即用。

```properties
# Linux
wget https://github.com/c-bata/kube-prompt/releases/download/v1.0.6/kube-prompt_v1.0.6_linux_amd64.zip
unzip kube-prompt_v1.0.6_linux_amd64.zip

# macOS (darwin)
wget https://github.com/c-bata/kube-prompt/releases/download/v1.0.6/kube-prompt_v1.0.6_darwin_amd64.zip
unzip kube-prompt_v1.0.6_darwin_amd64.zip

# 给 kube-prompt 加上执行权限并移动常用的可搜索路径。
chmod +x kube-prompt
sudo mv ./kube-prompt /usr/local/bin/kube-prompt
```

