# Noah provisioning role

Noah mode is an explicit layer on top of the existing `splunk.role`. It is
enabled only by `SPLUNK_NOAH_ENABLED=true`; adding a `[noahService]` stanza
does not activate this role.

| `splunk.role` | Noah client behavior | Heartbeats | Peer/bucket-map discovery |
| --- | --- | --- | --- |
| `splunk_indexer` | Peer | Enabled | Disabled |
| `splunk_search_head` | Search client | Disabled | Enabled |
| `splunk_deployer` | None | Disabled | Disabled |

Provisioning has two deliberate entry points:

1. `pre_auth.yml` writes a complete but disabled `[noahService]` stanza
   before docker-splunk starts its temporary authentication process.
2. `post_config.yml` selects exactly one role profile after declarative
   configuration is rendered and before the first full splunkd start.

Unsupported Splunk roles fail validation when Noah mode is enabled. With Noah
mode disabled, Noah client configuration is not written.

The SOK controller owns the shared `[noahService]` values such as `uri`,
`tenant`, and the desired enabled state.

Only indexers advertise an address. It resolves in this order: a non-empty
`SPLUNK_NOAH_ADVERTISED_ADDR`, then a non-empty defaults-file
`splunk.noah_advertised_addr`, then derivation from the Kubernetes pod identity
as `<scheme>://<POD_NAME>.<SPLUNK_HEADLESS_SERVICE_NAME>.<POD_NAMESPACE>.svc.<CLUSTER_DOMAIN>:<SPLUNK_SVC_PORT>`.
The scheme follows the splunkd management endpoint, so a non-TLS endpoint
advertises `http`. An unusable value, or a required identity variable that is
absent when derivation is needed, fails before splunkd starts. Search-head and
deployer stable identities remain separate settings.

The Noah `pass4SymmKey` is delivered
exclusively through `splunk.conf.server.content.noahService.pass4SymmKey`
(from a Kubernetes Secret); it is intentionally separate from
`splunk.pass4SymmKey`, which is the Splunk-to-Splunk `[general]` key.

Search-head cluster formation is deliberately shared rather than implemented
inside this Noah role. The common Linux SHC pre-start task writes the stable
member and replication configuration while splunkd is stopped for both classic
and Noah deployments. Classic mode also writes its Cluster Manager peering;
Noah mode writes only the Noah search-client settings here. The later SHC
commands form and verify the cluster without scheduling duplicate restarts.
