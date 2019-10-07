# ansible-vault lookup module [![Build Status](https://travis-ci.org/jhaals/ansible-vault.svg?branch=master)](https://travis-ci.org/jhaals/ansible-vault)
This is a lookup module for secrets stored in [HashiCorp Vault](https://vaultproject.io/).
Supports Ansible 1.9.x and 2.x

### Installation
lookup plugins can be loaded from several different locations similar to `$PATH`, see
[lookup_plugins](http://docs.ansible.com/ansible/intro_configuration.html#lookup-plugins).
An example setup can be found in the [tests](tests) directory.

The source for the plugin can be pointed to via a _requirements.yml_ file, and
accessed via [`ansible-galaxy`](http://docs.ansible.com/ansible/galaxy.html).

### Configuration
The address to the Vault server:

    export VAULT_ADDR=http://192.168.33.10:8200/

The plugin supports both Vault auth token and GitHub auth token. To use Vault auth token:

    export VAULT_TOKEN=56f48aef-8ad3-a0c4-447b-8e96990776ff

If your Vault server is configured to use GitHub auth token:

    export VAULT_GITHUB_API_TOKEN=56f48aef-8ad3-a0c4-447b-8e96990776ff

The plugin also supports Vault's CA-related environment variables, to
enable use of a server certificate issued by a not-widely-trusted
Certificate Authority. Use of this feature in the plugin requires
Python 2.7.9.

    export VAULT_CACERT=/etc/ssl/certs/localCA.pem
    export VAULT_CAPATH=/etc/ssl/localCA

The Vault address, CA certificate, and path can also be set via the Ansible
variables `vault_addr`, `vault_cacert`, and `vault_capath`, respectively.

    export VAULT_CAHOSTVERIFY="no"

This avoid the hostname check for Vault certificate (useful with self-signed certicates).
This option can also be set via the Ansible variable `vault_cahostverify`.

    export VAULT_SKIP_VERIFY=1

This will disable ssl certs validation. `VAULT_CACERT`, `VAULT_CAPATH` and `VAULT_CAHOSTVERIFY` have to be unset.
Can also be set via Ansible variable `vault_skip_verify: <bool>`.

For more information on setting variables in Ansible, see the
[variables docs](http://docs.ansible.com/ansible/playbooks_variables.html).

The Vault token intentionally can **not** be set via an Ansible variable, as
this is generally checked into revision control and would be a bad security
practice somewhat defeating the purpose of using Vault. The token can be read
from the file `$HOME/.vault-token`, as documented at
[Vault environment variables](https://www.vaultproject.io/docs/commands/environment.html).

If any such parameter is set by both an environment variable and an
alternative means, the environment variable takes precedence.

### Caching

By default secrets fetched from Vault will be cached in memory, unless you specify

    export ANSIBLE_HASHICORP_VAULT_USE_CACHE=no

Note that secrets will be fetched once per fork (defaults to 5). If you turn off
this feature by toggling above variable, all lookups will be done per node instead.

### [`Approle`](https://www.vaultproject.io/docs/auth/approle.html) support

If you want to use the Approle auth backend, you can do this by setting the follwing
environment variables. If those vars are set, it is tried to get an approle token. If
caching is enabled, the token is stored in the cache, so that it can be reused.

    export ANSIBLE_HASHICORP_VAULT_ROLE_ID=ba78195c-12c9-557f-f8e2-75705b9b52ec
    export ANSIBLE_HASHICORP_VAULT_SECRET_ID=5a4d079b-e6aa-ad54-8b0c-09dd35b740ee

Per default the authentication will be done against ${YOUR_HOST}/v1/auth/approle/login
You can change this to fit your layout by setting the following var to your value:

    export ANSIBLE_HASHICORP_VAULT_ROLE_PATH=v1/auth/my/role/path/approle/login

### Usage
ansible-vault works as any other lookup plugin.

```yaml
- debug: msg="{{ lookup('vault', 'secret/foo', 'value') }}"
```

```yaml
# templates/example.j2

# Generic secrets
{{ lookup('vault', 'secret/hello').value }} # world
# Generic secrets with parameters
{{ lookup('vault', 'pki/issue/example-dot-com common_name=foo.example.com format=pem_bundle').certificate }}
# Specify field inside lookup
{{ lookup('vault', 'secret/hello', 'value') }} # world
# This syntax for Ansible 1.9.x
{{ lookup('vault', ['secret/hello', 'value']) }} # world

# Dynamic secrets
{% set aws = lookup('vault', 'aws/creds/deploy') %}
access_key = {{ aws.access_key }} # AKSCAIZSFSYHFGA
secret_key = {{ aws.secret_key }} # 4XSLxDUS+hyXgoIHEhCKExHDGAJDHFiUA/adi
```

If the desired value is stored within Vault with the key 'value' (like
'value=world' shown above), within a task, the lookup can be performed with:

```yaml
with_vault:
- secret/hello
```

And then referenced with `"{{ item.value }}"`

Alternatively, the lookup can be performed with:

```yaml
with_vault:
- secret/hello
- value
```

And then referenced with `"{{ item }}"`

Both of these forms work with both Ansible 1.9.x and 2.x. They only work
within tasks, though. You can **not** use the `with_vault:` syntax within a
variable definition file.

### Troubleshooting

If you use the version of Ansible shipped with Ubuntu 16.04 (2.0.0.2-2ubuntu1) and get the error ```Error was sequence item 0: expected string, dict found``` then you should [install Ansible from the PPA](http://docs.ansible.com/ansible/latest/intro_installation.html#getting-ansible) instead. Related to [this Ansible PR](https://github.com/ansible/ansible/pull/14591)


### What's the difference between `ansible-vault` and [`hashi_vault`](https://github.com/ansible/ansible/blob/devel/lib/ansible/plugins/lookup/hashi_vault.py)

- (Ansible Vault) No external dependencies; (hashi_vault) requires hvac
- (Ansible Vault) Uses the same environment variables as vault itself
- (Ansible Vault) Quicker update cycle
- (Ansible Vault) Supports dynamic secrets
- (Ansible Vault) Supports custom fields
