# Molecule Testing

This role is tested with [Molecule](https://molecule.readthedocs.io/en/latest/) using the Docker driver. The
scenario lives in `molecule/default/`.

## Scenario layout

| File             | Purpose                                                                                  |
| ---------------- | ---------------------------------------------------------------------------------------- |
| `molecule.yml`   | Driver, platform image and test sequence configuration                                   |
| `prepare.yml`    | Prepares the test instance before the role is applied                                    |
| `converge.yml`   | Applies the role to the test instance                                                    |
| `verify.yml`     | Installs [Goss](https://github.com/goss-org/goss) and runs the tests defined in `tests/` |
| `tests/*.yml.j2` | Goss test definitions, templated per OS family                                           |

## How it works

- The platform is a `geerlingguy/docker-<distro>-ansible` image (default `ubuntu2604`, override with the
  `MOLECULE_DISTRO` environment variable), run privileged with `/sys/fs/cgroup` mounted so systemd works inside the
  container.
- `test_sequence` runs `converge` and `verify` twice in a row. The second pass checks that the role is idempotent
  and that it still passes verification after a no-op run.
- `verify.yml` downloads the latest Goss release, renders every `tests/*.yml.j2` template on the target host, and
  runs `goss validate` against the rendered files.

## Running tests

```sh
molecule test         # full lifecycle: create, prepare, converge, verify (x2), destroy
molecule converge     # apply the role without destroying the instance afterwards
molecule verify       # re-run the Goss checks against a running instance
molecule destroy      # tear down the instance
```

Test against a different distro:

```sh
MOLECULE_DISTRO=rockylinux9 molecule test
```

## Adding tests

Add or edit assertions in `tests/*.yml.j2` using [Goss syntax](https://goss.readthedocs.io/en/stable/gossfile/#important-note-about-goss-file-format)
(`service`, `file`, `command`, `port`, `package`, etc.). Templates have access to Ansible facts, so use
`ansible_facts['os_family']` to branch on distro-specific checks.
