# Exam Prep

Reminders
- 120 minutes
- Remote desktop
    - Work
    - Notes in vim
    - Base docs
    - API ref
- Checkin process
    - ID check and camera scan
    - Clean testing area
    - One monitor

Tips/tricks/alias
- https://killer.sh/
    - (exam simulator free if you bought it)
- https://killercoda.com/killer-shell-cka
- https://kubernetes.io/docs/home/

Copy/Paste
- Right click menu
- ctrl-shift-(c|v)
- One click copy from instructions (recommended to avoid typos)

Terminal
- `kubectl` pre-aliased to `k` with autocompletion
- `.vimrc` is pre-set `shiftwidth=2; expandtab=true; tabstop=2`
- vscode and webstorm available
    - https://docs.linuxfoundation.org/tc-docs/certification/lf-handbook2/exam-user-interface#virtual-machine-jsnad-and-jsnsd-exams-only

Finesse Guide
-------------

- Be imperative first, declarative second
    - `k create | run ... -h`
- Copying yaml from docs is LAST RESORT
    - Make thing with `k create|run`
    - Add `... --dry-run=client -o yaml > resource.yaml` if you need to add things before apply
- Use the `-h` option
    - `k create -h` to see what you can create
    - Tells you exactly what can be created imperatively WITH EXAMPLES
    - Menu increases in detail with base commands

Understand K8s docs
- Remember important pages/examples
    - mentally bookmark templates that can't be created interactively
        - pv, pvc, network policies
    - ctrl-f `kind: <MY-RESOURCE>`
    - Use 1 pager ref: https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.24/

Use shortnames
- Never type out full resource name if you can help
    - cm -> configmap
    - check out all shortnames with `k api-resources`

Aliases, functions, and variables

```
# MUST HAVES (issue on my comp)

## create yaml on the fly faster (dry output var)
export do="--dry-run=client -o yaml"

## organize files per question
mkcd() { mkdir -p "$@" && cd "$@" ; }
## mkcd 16 (for question 16)
```

```
# nice to haves

## create/destroy from yaml faster
alias kaf='k apply -f '
alias kdf='k destroy -f '

## namespaces
export nk='-n kube-system'
export n='-n important-ns' # set as needed

## destroy without waiting
export now='--grace-period 0 --force'
```
