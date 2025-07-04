# 🛠️ Helm Cheat Sheet for `tic-tac-toe`

## 🔄 Install or Upgrade

Install the app:

```bash
helm install tic-tac-toe helm/tic-tac-toe -n default
```

Upgrade (and install if not yet installed):

```bash
helm upgrade --install tic-tac-toe helm/tic-tac-toe -n default
```

Use custom values:

```bash
helm upgrade --install tic-tac-toe helm/tic-tac-toe -n default -f custom-values.yaml
```

---

## 📟 Inspect Releases

List all Helm releases in a namespace:

```bash
helm list -n default
```

Check the status of the release:

```bash
helm status tic-tac-toe -n default
```

Get values used in the release:

```bash
helm get values tic-tac-toe -n default
```

Get full manifest applied by Helm:

```bash
helm get manifest tic-tac-toe -n default
```

Save current values to a file:

```bash
helm get values tic-tac-toe -n default -o yaml > current-values.yaml
```

---

## 🧪 Test & Validate

Lint the chart (checks for errors):

```bash
helm lint helm/tic-tac-toe
```

Dry-run and preview generated YAML:

```bash
helm template tic-tac-toe helm/tic-tac-toe -n default
```

Diff changes before applying (requires plugin):

```bash
helm diff upgrade tic-tac-toe helm/tic-tac-toe -n default
```

Install the diff plugin (one-time):

```bash
helm plugin install https://github.com/databus23/helm-diff
```

---

## ❌ Uninstall

Remove the release and all resources it created:

```bash
helm uninstall tic-tac-toe -n default
```

---

## 🧠 Pro Tips

* Helm doesn’t delete manually created secrets like `github-container-registry`.
* Namespace is required if you're not using `default`.
* Use `--atomic` during `install` or `upgrade` to roll back on failure.

```bash
helm upgrade --install tic-tac-toe helm/tic-tac-toe -n default --atomic
```

---

## 🔒 One-time Setup for GHCR (if not done yet)

```bash
kubectl create secret docker-registry github-container-registry \
  --docker-server=ghcr.io \
  --docker-username=<your-username> \
  --docker-password=<your-token> \
  --namespace=default
```

---

Happy Helming! 🚀
