## Epic Description

The various `subctl` commands support a mixture of `KUBECONFIG` environment variables, `--kubeconfig` flags, `--kubecontext` flags,
kubeconfig files etc.

We should make this consistent, ideally using `clientcmd`’s tools to handle kubeconfig.
[Operator PR #23](https://github.com/skitt/submariner-operator/pull/23) has an old attempt at this,
using [`BindOverrideFlags()`](https://pkg.go.dev/k8s.io/client-go@v0.23.5/tools/clientcmd#BindOverrideFlags) to provide access
to all the features supported by `clientcmd` (see `kubectl options`).

## Acceptance Criteria

For commands using a single context, _all_ the following should work,
with kubeconfigs containing one or more clusters and/or contexts (this uses a `kube` prefix when setting up `clientcmd`):

```bash
# Use the default context in the kubeconfig
KUBECONFIG=/path/to/kubeconfig subctl foo
subctl foo --kubeconfig /path/to/kubeconfig
subctl --kubeconfig /path/to/kubeconfig foo

# Use the specified context from the kubeconfig
KUBECONFIG=/path/to/kubeconfig subctl foo --kubecontext bar
subctl foo --kubeconfig /path/to/kubeconfig --kubecontext bar
subctl --kubeconfig /path/to/kubeconfig --kubecontext bar foo
```

For commands using multiple contexts, the prefixes should make the purpose of each context clear,
using appropriate prefixes when setting up `clientcmd`:

```bash
KUBECONFIG=/path/to/kubeconfig subctl benchmark latency --fromcontext foo --tocontext bar
subctl benchmark latency --kubeconfig /path/to/kubeconfig --fromcontext foo --tocontext bar
```

`--kubeconfig` is preserved as-is, since users can use a single file combining all their contexts.
However such commands should also support separate kubeconfigs:

```bash
subctl benchmark latency --fromconfig /path/to/fromconfig --toconfig /path/to/toconfig
```

`clientcmd` _doesn’t_ handle `--kubeconfig` itself, that needs to be handled by `subctl`.

This enhancement should also include a test suite covering all the different possibilities.
As far as possible, the 0.12 options should keep their existing behaviour, and be marked for deprecation;
they will be removed after 0.13.

This would subsume [`--kubecontexts` support in `subctl diagnose`](https://github.com/submariner-io/submariner-operator/issues/1327).

## Definition of Done (Checklist)
<!-- Make sure to check all relevant items before end of the release: -->

* [ ] Code complete
* [ ] Relevant metrics added
* [ ] The acceptance criteria met
* [ ] Unit/e2e test added & pass
* [ ] CI jobs pass
* [ ] Deployed using cloud-prepare+subctl
* [ ] Run subctl verify, diagnose and gather
* [ ] Uninstall
* [ ] Troubleshooting (gather/diagnose) added
* [ ] Documentation added
* [ ] Release notes added

## Work Items

* [ ] Create a test suite for the various `--kubeconfig` and `--kubecontext` variants
* [ ] Delegate flag setup for `--kubecontext` and related flags to `k8s.io/client-go/tools/clientcmd`
* [ ] Integrate clientcmd handling into `restconfig.Producer`
* [ ] Where appropriate, add `--kube` variants for command which need multiple contexts

It will probably be necessary to isolate the `clientcmd` data structures in each command using them,
to ensure that uninitialised data structures can’t be accidentally used.
For example, a command using `--from` and `--to` instead of `--kube` shouldn’t end up with unused
`--kube` flag variables.
See [`kubectl apply`](https://github.com/kubernetes/kubectl/blob/master/pkg/cmd/apply/apply.go) for
inspiration.

This work will flip the way `clientcmd` settings are handled.
Each command using `clientcmd` will need to maintain a `clientcmd.ConfigOverrides` instance as part
of its state (_e.g._ in a structure describing its settings), and “hook up” the various flags to
fields in that structure: `--kubeconfig` maps to `clientcmd.DefaultClientConfig.ExplicitPath`,
the remaining flags are set up with `clientcmd.BindOverrideFlags` using an appropriate prefix
(`kube` in most cases, so the flags end up being `--kubecontext` etc.).
The plural form, `--kubecontexts`, will need special handling.

The following illustrates the initial setup:

```go
// addKubeContextFlag adds a "kubeconfig" flag and a single "kubecontext" flag that can be used once and only once
func addKubeContextFlag(cmd *cobra.Command) {
    loadingRules := clientcmd.NewDefaultClientConfigLoadingRules()
    loadingRules.DefaultClientConfig = &clientcmd.DefaultClientConfig
    overrides := clientcmd.ConfigOverrides{}
    kflags := clientcmd.RecommendedConfigOverrideFlags("kube")
    addKubeConfigFlag(cmd, &loadingRules.ExplicitPath)
    clientcmd.BindOverrideFlags(&overrides, cmd.PersistentFlags(), kflags)
    clientConfig = clientcmd.NewNonInteractiveDeferredLoadingClientConfig(loadingRules, &overrides)
}
```
