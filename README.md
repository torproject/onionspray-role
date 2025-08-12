# Onionspray

This role clones the [Onionspray][] repo and builds from source the necessary
software (OpenResty with the Nginx [http_substitutions_filter][] module,
[Onionbalance][], Tor).

[Onionspray]: https://onionservices.torproject.org/apps/web/onionspray/
[Onionbalance]: https://onionservices.torproject.org/apps/base/onionbalance/
[http_substitutions_filter]: https://github.com/yaoweibin/ngx_http_substitutions_filter_module

The role first generates the configuration files needed to serve a website. The
build is then done by the `opt/build-DISTRO.sh` script inside the Onionspray
repo, executed by this role, depending on the distribution of your server and
if supported.

## Requirements

The remote host should have `git` installed prior to running the role. The
other necessary packages are installed by the build process of Onionspray.

Your Ansible controller should use pipelining. In your `ansible.cfg`:

```
[connection]
pipelining=True
```

## Quick-start

Assuming you have a host named `myhost` on which you can run
`ansible-playbook`, and you cloned this role in a `roles` directory, this is an
example of a basic playbook:

```
- name: Onionspray Tor proxy
  hosts: myhost
  roles:
    - onionspray
```

You can configure your project(s), i.e. the website(s) that Onionspray will
handle, using the `onionspray_project_settings` variable, a list of
dictionaries. You may also want to (re)define other values: check below for a
complete list of variables and their usage.

As an example, you could have in `host_vars/myhost.yml`:

```
onionspray_repo_git_revision: a0e43045fe135e1b3f5b96e075ed519e4359ab7f

onionspray_project_settings:
  - project_name: "example1"
    softmaps:
      - proxied_domain: example.com
      - proxied_domain: example.org
  - project_name: "example2"
    hardmaps:
      - tor_address: yetkvkuqlr23sdzkf2mynt7aixfjzq6pjys2ffurr3hzpyfxrc7swpqd
        proxied_domain: example.net
    foreingmaps:
      - tor_address: exaymhwjhgdopeebkv5p3lmb5vu2mmvcc7krpgwx6ngf4uvob2whkcqd.onion
        proxied_domain: example.info
    log_separate                 : '1'
    nginx_resolver               : '127.0.0.53 ipv6=off'
    nginx_cache_seconds          : '60'
    nginx_cache_size             : '64m'
    nginx_tmpfile_size           : '8m'
    x_from_onion_value           : '1'
    tor_export_circuit_id        : 'haproxy'
    tor_intros_per_daemon        : '6'
    tor_single_onion             : '1'
    tor_pow_enabled              : '1'
    tor_max_streams              : '5000'
    tor_max_streams_close_circuit: '1'
    tor_intro_dos_defense        : '1'
    tor_intro_dos_burst_per_sec  : '20000'
    tor_intro_dos_rate_per_sec   : '20000'
    project_custom_settings: |
      # block access to "forbidden" subdomain
      set block_err This subdomain is forbidden.
      set block_host_re ^forbidden\.
    project_custom_settings: |
      # block access to "forbidden" subdomain
      set block_err This subdomain is forbidden.
      set block_host_re ^forbidden\.

onionspray_keys:
  - public_key_base64: BASE64_ENCODED_PUBLIC_KEY
    secret_key_base64: BASE64_ENCODED_SECRET_KEY
    tor_address: yetkvkuqlr23sdzkf2mynt7aixfjzq6pjys2ffurr3hzpyfxrc7swpqd
```

## Self-signed certificate

This role uses the Onionspray's default certificate, which is a generated
self-signed certificate. The role provides variables to change the fields,
documented below.

In Tor, all requests are encrypted by the protocol. The URL itself is the
guarentee that you are connecting to the right server. It is hence not strictly
necessary to generate a valid HTTPS certificate, [more info
here](https://community.torproject.org/onion-services/advanced/https/).

However, it is still better to use a valid HTTPS certificate, to avoid HTTPS
warnings on browsers such as Brave for example. The Tor Browser does not
display HTTPS warnings if using a self-signed certificate with an Onion
service, though this may change in the future.

Should you want to get a valid HTTPS certificate, both
[HARICA](https://blog.torproject.org/tls-certificate-for-onion-site) (normal
certificate, $10/year) and
[Digicert](https://www.digicert.com/blog/onion-officially-recognized-special-use-domain/)
(expensive, EV certificate) provide them. Let's Encrypt and other providers
using the ACME protocol (i.e. automation possible through `certbot` for
example) still do not support these certificates. Using these valid
certificates is hence a manual operation, as long as [this
standard](https://acmeforonions.org/) is not implemented.

## Variables

### `onionspray_build_lock_file`

File indicating whether Onionspray has already been built or not. If the file
exists, the build process is skipped. Defaults to `{{
onionspray_repo_download_path }}/onionspray-already-built.lock`.

### `onionspray_build_script_name`

The script from the Onionspray repo to use when building it. If not specified,
the role will try to find a script from the host's distribution and release.
You can use this variable to specify which script to use, the list [is
available in the Onionspray
repo](https://gitlab.torproject.org/tpo/onion-services/onionspray/-/tree/main/opt?ref_type=heads).
For example, `build-centos-8.2.2004.sh`.

### `onionspray_ca_file`

The path of a file containing one or more certificate authorities (CAs), that
will be used by Onionspray to validate the TLS certificates of the proxied
domains. By default, contains the path of the CA file managed by the Debian
`ca-certificates` package (`/etc/ssl/certs/ca-certificates.crt`).

The role will abort if this file does not exist. If you wish to skip this
verification (not recommended, as you expose your Onionspray host to
Man-In-The-Middle attacks if you do so), you can redefine the
`onionspray_check_cert_with_ca_file` variable.

### `onionspray_check_cert_with_ca_file`

A variable controlling whether or not to check the TLS certificate of the
domain your Onionspray host is proxying, against a certificate authority
defined in the `onionspray_ca_file`. Defaults to `true`.

### `onionspray_keys`

A list of public/secret keypairs, and their corresponding `.onion` v3 address.
Defining these keys is necessary to persist a given `.onion` address, and use
it in hardmaps or softmaps (check the `onionspray_project_settings` variable for
more information).

Each list item should define the three following fields. All of them are
mandatory.

#### `public_key_base64`

The Onion v3 public key to use, encoded in Base64.

The Base64-encoded key can be produced directly from a generate public key,
with

    cat hs_ed25519_public_key | base64

#### `secret_key_base64`

The Onion v3 secret key to use, encoded in Base64.

It's recommended that this variable to be also encrypted using [Ansible Vault][].

[Ansible Vault]: https://docs.ansible.com/ansible/latest/vault_guide/vault.html

The Base64-encoded and then encrypted key can be produced directly from a
generate public key, with

    cat hs_ed25519_secret_key | base64 | ansible-vault encrypt_string

#### `tor_address`

The corresponding `.onion` v3 address to the defined public and secret keys.

### `onionspray_project_settings`

A list of dictionaries, each dictionary representing a project (i.e. a clearnet
website that Onionspray will proxy). The supported fields per dictionary are
listed below.

#### `project_name`

REQUIRED: the name of your Onionspray project, used to set the configuration
and settings filenames.

#### `hardmaps` and `softmaps`

These two fields are used to define the hardmaps or softmaps used by
Onionspray. You can find more information on these concepts in the [Onionspray
documentation](https://onionservices.torproject.org/apps/web/onionspray/guides/balance/).

They support the same fields.

##### `proxied_domain`

REQUIRED: the clearnet domain name to which Onionspray will proxy requests.

##### `tor_address`

The custom `.onion` v3 address to use, dependent on the public/secret keys
used. The corresponding public/secret keys, using the same `tor_address` value,
should be defined in the `onionspray_keys` variable to use this field.

If this field and the public/secret keys in the `onionspray_keys` variable are
not defined, a new keypair will be generated. Keep in mind that if you modify
the project configuration without defining this field, a new keypair will be
generated in the next Ansible run, and so the `.onion` address used by
Onionspray will change.

#### `foreignmaps`

The `foreignmaps` variable is used to store onion-to-site mappings that exist
outside of this particular configuration file, eg: for some other sites.

Each entry should use the following fields.

##### `proxied_domain`

The clearnet domain name to which Onionspray will proxy requests.

##### `tor_address`

The Onion Service address for the foreign map.

#### `nginx_*` and other proxy settings

This role supports the following proxy-related settings from [Onionspray][]:

* `x_from_onion_value`.
* `inject_headers_upstream`.
* `log_separate`.
* `nginx_resolver`.
* `nginx_tmpfile_size`.
* `nginx_cache_seconds`.
* `nginx_cache_size`.
* `nginx_x_onion_circuit_id`.

#### `tor_*` settings

The `tor` daemon used for each project can be customized using [Onionspray][]
settings:

* `tor_export_circuit_id`.
* `tor_intros_per_daemon`.
* `tor_single_onion`.
* `tor_pow_enabled`.
* `tor_pow_queue_rate`.
* `tor_pow_queue_burst`.
* `tor_intro_dos_defense`.
* `tor_intro_dos_burst_per_sec`.
* `tor_intro_dos_rate_per_sec`.
* `tor_max_streams`.
* `tor_max_streams_close_circuit`.

#### `project_custom_settings`

If additional settings are neded (those not explicitly supported by this role),
you can put them in this variable and they will be appended to the generated
`PROJECT_NAME.conf` configuration file.
Defaults to an empty string.

### `onionspray_repo_url`

The URL of the Onionspray repo to clone, defaults to [the official Onionspray
repo](https://gitlab.torproject.org/tpo/onion-services/onionspray/).

### `onionspray_repo_download_path`

Where to put the cloned Onionspray repository, defaults to `{{
onionspray_user_homedir }}/onionspray`.

### `onionspray_repo_git_revision`

The revision of the repository to checkout. No default value: if not defined,
the role uses the latest revision of the main branch.

### `onionspray_selfsigned_cert_lifetime_days`

Lifetime of the self-signed certificate generated for the .onion website,
defaults to 365 days (`365`).

### `onionspray_selfsigned_cert_country`

Self-signed certificate country name. Defaults to `AQ` (Antartica).

### `onionspray_selfsigned_cert_locality`

Self-signed certificate locality. Defaults to `Onion Space`.

### `onionspray_selfsigned_cert_organization`

Self-signed certificate organization. Defaults to `The SSL Onion Space`.

### `onionspray_selfsigned_cert_organizational_unit`

Self-signed certificate organizational unit. Defaults to `Self Signed
Certificates`.

### `onionspray_selfsigned_cert_state_or_province`

Self-signed certificate state or provice name. Defaults to `The Internet`.

### `onionspray_use_systemd`

Controls whether systemd is used: set to `false` e.g. when running CI jobs in
environments without systemd. Defaults to `true`.

### `onionspray_user`

System user that runs Onionspray, will be created. A group of the same name is
automatically created. Defaults to `onionspray`.

### `onionspray_user_homedir`

The home directory of the user running Onionspray. Defaults to `/home/{{
onionspray_user }}`.

## Contributing

All contributions are very welcome. Feel free to send your enhancements and
patches as PRs, or open issues.

## Development

This role has [molecule tests](molecule):

* The `podman` scenario is a generic one and is well suited for testing both
  locally and through CI.
* The `local` scenario actually applies the configuration into the running
  node, so be careful were to run it.

A [Makefile](Makefile) exists to help local testing, which relies on
[AnCIble][] to be available somewhere. Details in how to use it are given
[here][].

[AnCIble]: https://gitlab.torproject.org/tpo/onion-services/ansible/ancible/
[here]: https://gitlab.torproject.org/tpo/onion-services/ansible/ancible/-/blob/main/README.md#development

## License

This project is licensed with the Affero GPLv3. Check [LICENSE](LICENSE) for the
full license, or [this page](https://choosealicense.com/licenses/agpl-3.0/) for
a quick recap. In general, if you use a modified version of this role, you must
make the source code public to comply with the AGPL.

## Acknowledgements

Many thanks to [Mediapart](https://www.mediapart.fr) for which this role has
been created, for allowing it to be open sourced. You can visit their website
over Tor at
[https://www.mediapartrvj4bsgolbxixw57ru7fh4jqckparke4vs365guu6ho64yd.onion/](https://www.mediapartrvj4bsgolbxixw57ru7fh4jqckparke4vs365guu6ho64yd.onion/).

## References

* The Onionspray documentation: [quickstart
  guide](https://onionservices.torproject.org/apps/web/onionspray/tutorial/),
  [troubleshooting](https://onionservices.torproject.org/apps/web/onionspray/guides/troubleshooting/)
  sections mainly.
* Great blogpost: [A Complete Guide to EOTK](https://shen.hong.io/making-websites-on-tor-using-eotk/)
* Another great blogpost: [ProPublica's experience with
  EOTK](https://www.propublica.org/nerds/a-more-secure-and-anonymous-propublica-using-tor-hidden-services)
