# csync-repo
Demonstration repo for use wth Google Config sync

TO DO write up install and put demo together

307  kubectl apply -f config-sync-operator.yaml

  310  vi config-management.yaml
  311  ls -la
  312  kubectl apply -f config-management.yaml
  313  git clone https://github.com/StuartCTS/csync-repo.git
  314  cd csync-repo
  315  ls -la
  316  code .
  317  cd ../..
  318  ls -la
  319  cd config-sync
  320  nomos
  321  nomos status
  322  pwd
  323  ls -la
  324  cd csync-repo
  325  nomos init
  326  nomos init --force
  327  ls -la
  328  cat README.md
  329  kubectl get all -n shipping-dev
  330  ls -la
  331  cd ..
  332  ls -la
  333  bat config-management.yaml
  334  kubectl get resource-quota -n shipping-dev
  335  kubectl get resourcequota -n shipping-dev
  336  kubectl get resourcequota -n shipping-dev
  337  tree
  338  history