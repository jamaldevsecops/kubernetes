1. Open the bash profile. 
```
vim ~/.bashrc
```
2. Past the following lines and save & exit. 
```
#------------------------------------------kubernetes--------------------------------------#
source <(kubectl completion bash)
complete -F __start_kubectl k
alias k=kubectl
alias kn='k config set-context --current --namespace'
```
3. Reload the bash profile.
```
source ~/.bashrc
```
4. Now try it.
```
kub<tab> get pod<tab><tab>
```
