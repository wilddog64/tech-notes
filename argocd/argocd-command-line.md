# argocd command-line tips and tricks

## Will output `-o json` better than `-o name`?

The short answer is no. The `-o` doesn't change what cross the wire. The argocd cli taks to the API server over gRPC/protobuf; the `-o json|yaml|name|wide` flag only affects local printing after response arrives. So JSON won't make the network heavier for X number of apps vs YAML -- same plyload from the server.

if you really want to cut network/CPU for big fleets, reduce what the server returns, consider the following steps,

* Filter at server-side by

  `-p/--project, -l/selector, -r/--repo, -c/--cluster, -N/--app-namespace`. These constrains are sent with the request so that server sends fewer app back. For example,

  ```bash
  argocd app list -p default -l 'env in(prod,stage)' -o json | jq -r '.[].metadata.name'
  ```

* If you only need names and don't need Argo CD's computed status, hit Kubernetes directly:

  ```bash
  kubectl get applications.argoproj.io -n argocd -o name
  ```
  or
  ```bash
  kubectl get applications.argoproj.io -A -o name | cut -d/ -f2
  ```
  This bypasses the Argo CD API.

* You can also run the CLI in "core" mode (talks straight to the custer instead of argocd-server), but you will miss some Argo CD server features/status: `argocd list --core`.

## Avoid placing argocd command line call in a loop

Looping `argocd` calls is a heavy toll. For better pattern,

* Fetch once, reuse locallly (no per-app RPCs):

  ```bash
  json=$(argocd app list -o json)
  mapfile -t apps < <(jq -r '.[].metadata.name' <<<"$json")
  # use $json for any filtering; use $apps only when you must call per app commands
  ```

* Batch instead of loop when possible:

  ```bash
  # one RPC against many apps by label
  argocd app sync -l 'env=prod'

  # or multiple explicit apps in on go
  argocd app sync appA appB appC
  ```

* When you must do per-app RPCs, parallelize carefully:

  ```bash
  printf '%s\n' "${apps[@]}" | xargs -n1 -P10 -I{} argocd app wait {} --timeout 300
  ```
  Avoid --hard-refresh unless required, and tune -P to your cluster.

* Network path tips:
  the CLI command talks to the server via gRPC; output flags only change local printing no thte payload. Use server-side filter to really shrink work. If latency is high, try `--port-forward` so the CLI connects iside the cluster.

* Get only a piece of app manifest:

  ```bash
  argocd app get-resource myapp --kind Ingress --filter-fields spec.rules,status.loadBalancer -o yaml
  ```

## Take avantage of git repo

1. Pull the mapping once from Argo CD: app -> repoURL, path, and commit
2. Mirror-clone repo and then remote update (cheap)
3. Read the file you need at the commit with git show and parse locally by jq

```bash
#!/usr/bin/env bash
set -euo pipefail

PROJECT="${PROJECT:-myproj}"            # argocd project filter
VALUES_FILE="${VALUES_FILE:-values-qa1.yaml}"  # which values file to read
CACHE_DIR="${CACHE_DIR:-$HOME/.cache/argocd-git-mirrors}"
JQ_FILTER='.items[]
  | {
      name: .metadata.name,
      repo: .spec.source.repoURL,
      path: .spec.source.path,
      # prefer the exact synced commit if present
      rev: (.status.sync.revision // .spec.source.targetRevision),
      # optional: helm valueFiles if you want to be precise
      valueFiles: (.spec.source.helm.valueFiles // [])
    }'

mkdir -p "$CACHE_DIR"

# 1) Get app -> repo/path/revision once
apps_json="$(argocd app list -p "$PROJECT" -o json)"

# 2) Clone/update each unique repo as a bare mirror
printf '%s\n' "$apps_json" \
| jq -r "$JQ_FILTER | .repo" \
| sort -u \
| while read -r repo; do
  [ -z "$repo" ] && continue
  key="$(printf '%s' "$repo" | sed 's#[^A-Za-z0-9._-]#_#g')"
  dir="$CACHE_DIR/$key.git"
  if [ ! -d "$dir" ]; then
    git clone --mirror "$repo" "$dir" >/dev/null
  else
    git --git-dir="$dir" remote update -p >/dev/null
  fi
done

# 3) For each app, cat the file at the commit and extract what you need
#    (example: print image.tag from the chosen values file)
printf '%s\n' "$apps_json" \
| jq -r "$JQ_FILTER | [ .name, .repo, .path, .rev, (if (.valueFiles|length)>0 then (.valueFiles|join(\"|\")) else \"\" end) ] | @tsv" \
| while IFS=$'\t' read -r name repo path rev value_files; do
  key="$(printf '%s' "$repo" | sed 's#[^A-Za-z0-9._-]#_#g')"
  dir="$CACHE_DIR/$key.git"

  # pick file: use Helm valueFiles if present, else VALUES_FILE
  files=()
  if [ -n "$value_files" ]; then IFS='|' read -r -a files <<< "$value_files"; else files=("$VALUES_FILE"); fi

  for vf in "${files[@]}"; do
    file_path="${path%/}/$vf"
    if content="$(git --git-dir="$dir" show "$rev:$file_path" 2>/dev/null)"; then
      # Prefer yq if available; else quick regex fallback
      if command -v yq >/dev/null; then
        tag="$(printf '%s' "$content" | yq -r '.image.tag // empty')"
      else
        tag="$(printf '%s' "$content" | perl -ne 'print "$1\n" if /^\s*tag:\s*"?([^"#\s]+)"?/ and $seen++==0')"
      fi
      echo -e "${name}\t${vf}\t${tag}"
    fi
  done
done | column -t
```

note:

* Prefer `.status.sync.revision` (exact commit Argo actually used) over `targetRevision`, which can be a branch/tag)
* If your apps use multi-source (`.spec.sources[]`), iterate those similarly
* For non-Helm stacks, swap the extraction step (e.g. `kustomization.yaml` fields, or run `helm template/kustomize build`locally if you truly need rendered output -- but that's still loal and parallelizable)

## If you only need tags

If all you need is a image tag, then here some fast, practical recipes:

* Rip through checked-out repo (regex; no YAML parser)

  ```bash
  # scan all values*.yaml files, 8-way parallel
find /path/to/repos -type f -name 'values*.y*ml' -print0 \
| xargs -0 -n1 -P8 perl -0777 -ne '
  # .image.tag: foo
  while (/^\s*tag:\s*["\x27]?([^"#\s]+)["\x27]?\s*$/mg) { print "$ARGV\t$1\n" }
  # image: repo/name:tag   (grab the tag part)
  while (/^\s*image:\s*[^:#\s"]+:([^#\s"]+)/mg)         { print "$ARGV\t$1\n" }
'
  ```

  Tips:
  * Want only tracked files? `git -C <repo> ls-files -z -- "values*.y*ml" | xargs -0 -n1 P8 ...`
  * Swap `-P8` to tune parallelism

## Safer parse with `yq` (still fast; handles YAML quirks)

  ```bash
  # handles either .image.tag or a string image: repo:tag
yq -r '
  .. | select(type=="map" and has("image")) |
  [input_filename, (.image.tag // (.image | select(type=="string") | split(":")[-1]))]
  | select(.[1] != null) | @tsv
' /path/to/repos/**/values*.y*ml
  ```
  Tips:
  If you need tags at the synced revision, do one `argocd app list -o json` to get `{repo, path, revision}`, mirrow repo, then read files locally:

  ```bash
  git --git-dir ~/.cache/mirrors/myrepo.git show <REV>:<path>/values-qa1.yaml | yq -r '.image.tag'
  ```
  run those `git show` call via `xargs -P` to parallelize -- still O(1) Argo calls

## Oneliner call with parallelizm

```bash
find . -name 'values-qa1.yaml' -type f -print0 \
| xargs -0 -n32 -P8 perl -nle 'print "$ARGV\t$1" if /^\s*tag:\s*["\x27]?([^"#\s]+)["\x27]?\s*(?:#.*)?$/'
```

* The above oneliner will batch 32 files per Perl process, up to 8 processes

## jq magic

Use a signle `argocd app list` call and reach each app's summary images -- no loop needs

```bash
argocd app list -o json \
| jq -r '
  (if type=="object" and .items then .items else . end)   # 1
  | .[]                                                   # 2
  | .metadata.name as $app                                # 3
  | (.status.summary.images // [])[]                      # 4
  | select(test(":") and (contains("@sha256:")|not))      # 5
  | capture("^(?<repo>.*?):(?<tag>[^:@]+)$")              # 6
  | "\($app)\t\(.tag)"                                    # 7
'
```
1. Normalize the input shape
   `argocd app list -o json` can return either an objec with `.items` or a bare array
   This line picks `.iterms` if present, otherwise uses the value as-is
2. Iterate apps
   `| .[]` iterates each Application
3. Save the app name
   Stores the current app's name in `$app` for later used
4. Iterate images
   Takes `.status.summary.images` (an array of strings like `repo/name:tag`) or `[]` if it's missing, and loops each image
5. Filter out non-tags
   Keeps only strings that contain `:` and do not contain `@sha256:` (digest pins)
6. Extract the tag with a regex
   `capture("^(?<repo>.*?):(?<tag>[^:@]+)$") splits at the final `:` because:
   * `.*` is greedy and the pattern is anchored to the end(`$`) (i.e. it's a normal tag, not a digest)
   * `[^:@]+` ensures the trailing piece has no`:` or `@` (i.e., it's a normal tag, not a digest)
   The result is an object like `{repo: "ghcr.io/foo/bar", tag: "1.2.3"}
7. Print app and tag: Formats as `APP<TAB>APP`

## An easy to understand python version

```bash
argocd app list -o json | python3 - <<'PY'
import sys, json
data = json.load(sys.stdin)
apps = data.get('items', data)
for app in apps:
    name = app['metadata']['name']
    for img in (app.get('status',{}).get('summary',{}).get('images') or []):
        img = img.split('@',1)[0]
        if ':' in img:
            print(f"{name}\t{img.rsplit(':',1)[1]}")
PY
```

note: if you want narrow down result set, use `-p/--project, -l/selector, -r/--repo, -c/--cluster, -N/--app-namespace` to filter at server side

## Some notes about xargs -n and -P

### -n and -P short explanation
* `-n N` pass `N` input items per command invocation
* `-P N` run up to `N` processes in parallel
* Together: xargs make batches of size N, then keeps up to P of those batches running concurrently. When one finishes, it starts the next

Example:
```bash
printf '%s\n' a b c d e f g h | xargs -n1 -P6 mycmd
```
Eight inputs -> 8 invocations total, ≤ 6 live at any moment

Key details and pitfalls

* Order: with `-P > 1`, completion order is `not` preserved (outputs can interleave)
* Batching: `-n32 -P8 cmd` = up to 8 processes, each ith 32 args
* Unlimited: GNU xargs allows `-P 0` = as many as possible
* Delimiters: use `-0` (and `find -print0`) for path with spaces/newlines
* Placeholders: if you need to insert the arg somewhere other than the end, use `-I{}` (GNU) or `-J{}` (BSD/macOS), or the robust pattern:
  ```bash
  xargs -n1 -P6 -I{} sh -c 'script "$1"' _ {}
  ```
### Custom scripts

Your custom script need to:

* xargs handles __launching__ concurrency; your script must be safe to run multiple times at once
* Pay ttention if the script:
  * rites to the same files/log (use unique, `mktemp`, or `flock`)
  * update shared state (DB transactions/locks)
  * prints to stdout (outputs will interleave; prefix lines or capture per-invication)
* If none of the above, you usually don't need to add parallel logic inside the script

Handy patterns

```bash
# Prefix outputs so you can sort later
seq 1 10 | xargs -n1 -P4 -I{} bash -c 'echo "start {}"; sleep $((RANDOM%3+1)); echo "done {}"' | sed -e 's/^/[job]/'

# Write per-arg logs (avoids interleaving)
printf '%s\0' file1 file2 | xargs -0 -n1 -P6 -I{} bash -c 'process "{}" >"{}.out" 2>"{}.err"'

# Serialize a critical section inside the script
# bash example:
{
  flock 9
  # critical work here
} 9>"/tmp/mylock"
```

Exit status (GNU xargs, userful for CI):
* `0` all commands succeeded;
* `123` some invocations failed (exit 1-125);
* `124` cmd line too long;
* `125` couldn't run;
* `126/127` not execuable/not found;
* `137` killed by signal 9 (e.g. out of memory)

How can a custom sript to handle -n8 from xargs

The bash script should iternate over `"$@"` (the arguments passed to that invocatioin):

```bash
#!/usr/bin/env bash
set -euo pipefail
for app in "$@"; do
  echo "processing $app"
  # do work per $app here
done
```

Then call it like:

```bash
printf '%s\n' app1 app2 app3 app4 app5 app6 app7 | xargs -0 -n8 -P6 <custom-script>
```
Common gotchas

* Don't use `-I{}` if you want batching as `-I` will forces one iterm per invocation
* Use `-0` + `find -print0` (or`printf '%s\0`) for paths with spaces/newlines
* If you need to insert args not `at the end`, wrap with `bash -c`:
  ```bash
  … | xargs -0 -n32 -P6 bash -c 'mycmd --flag "$@"' _
  ```
* If your script writes shared files/logs, ensure `concurrency safty` (`mktemp`, unique filenames, or `flock`). Outputs from parallel jobs may interleave on stdout -- prefix lines or write per job logs if needed
