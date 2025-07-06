---
title: 'Using Systemd Credentials for Secret Management in NixOS'
date: null
draft: true
---

# Secret Management in NixOS

The NixOS wiki has an [article on secret management libraries][nixos-secrets],
all of which have good support for asymmetric encryption.
I'm most familiar with [sops-nix][sops-nix] so that is what I will be using as
a comparison, but [agenix][agenix] and the others have the same workflow.

1. Create public/private key pairs for each of the devices you want to have
access to the secrets.
This includes keys for the system you use for development.

2. Add the public keys to a file in your repo.
For [sops-nix][sops-nix] this is `.sops.yaml`,
for [agenix][agenix] this is a Nix file such as `secrets.nix`.

3. Create a secret.
This involves writing the secret in some manner (via [`sops`][sops] or [`agenix`][agenix]),
then encrypting the data with a symmetric scheme (usually AES) with a randomly
generated encryption key.
That generated encryption key is then encrypted with the corresponding public
keys from step 2, and are stored alongside the encrypted message.

4. Distribute and unpack the secrets.
The file(s) containing the encrypted secrets are decrypted during system
startup, and loaded onto a directory on a `tmpfs`, normally under `/run`.
These files can have their permissions adjusted if needed.

This works well for 90% of usecases, and I would recommend using one of these
methods, especially since the support is not quite there for what I will
go over here.
Even so, there are some areas for improvement.

1. Secret permissions are assigned per-user, rather than per-service.
Most secrets you will be distributing will be used by a single service,
so managing access via file permissions may not be as fine-grained as you
would like.
It is also not compatible with `DynamicUser`, a setting that can improve
isolation of services.
This can be solved with the `LoadCredentials` unit option, which I will discuss
in the next section.

2. The master key is not as secure as it could be.
With [sops][sops], the developer's master key is a single point of failure
for an entire deployment.
If you are using [agenix][agenix] or [sops-nix][sops-nix] with PGP, age, or SSH
keys then the decryption key lives in accessible permanent memory.
Unless you have full-device encryption, a malicious actor with physical access
could read these secrets.
Or even simpler: an untrusted program can read the contents of
`~/.config/sops/age/keys.txt` or `~/.ssh`.
This can be solved with `LoadCredentialsEncrypted` and `systemd-creds`.

## Per-service secrets with `LoadCredentials`

As mentioned previously, secrets are usually associated with services and not

services with `DynamicUser=true` can't be granted
access secrets through file permissions.
Systemd has the option `LoadCredentials` to bridge the gap:

```nix
{ config, ... }: {
  # This will create /run/secrets/smtp_password, owned by root with 0400 permissions
  # The contents will be initialized from the default SOPS secrets file
  # The path is referenced through `config.sops.secrets.<name>.path`.
  sops.secrets.smtp_password = {};

  systemd.services.my-service = {
    description = "An example service";
    script = ''
      echo "Hello from my-service"
      echo "The secret file is stored at $CREDENTIALS_DIRECTORY/my-smtp-pass"
      echo "The secret is $(cat "$CREDENTIALS_DIRECTORY/my-smtp-pass")"
    '';
    serviceConfig = {
      LoadCredential = ["my-smtp-pass:${config.sops.secrets.smtp_password.path}"];
    };
    unitConfig = {
      AssertCredential = ["my-smtp-pass"];
    };
  };
}
```

```sh
$ sudo systemctl start my-service
$ journalctl -e -u my-service
The secret file is stored at /run/credentials/my-service.service/my-smtp-pass
The secret is MY_SECRET
```

In its simplest form, the `LoadCredential` setting takes a name and a file path
and loads the file contents into `/run/credentials/<service_name>/<secret_name>`,
setting the permissions to make it readable by the service.
The `$CREDENTIALS_DIRECTORY` environment variable is set to
`/run/credentials/<service_name>` to make it easier to reference,
as the path depends on the name of the service file.

Normally a missing credential is not considered an error.
If the path referenced by `LoadCredential` is missing, the credential is not
loaded, but the service is started regardless.
To force a failure at startup, the `AssertCredential` unit configuration
setting can be used.
There is also `ConditionCredential` to silently skip the service if the
credential is missing.

## Activation-Time Decryption with `LoadCredentialsEncrypted`

[nixos-secrets]: https://nixos.wiki/wiki/Comparison_of_secret_managing_schemes "NixOS Wiki: Comparison of secret managing schemes"
[agenix-gihub-issue]: https://github.com/ryantm/agenix/issues/123 "agenix: Integration with systemd-creds"
[nixcon-youtube]: https://www.youtube.com/watch?v=YFXwV0ZO9NE "NixCon: We should manage secrets the systemd way!"
[sops]: https://github.com/getsops/sops "sops"
[sops-nix]: https://github.com/Mic92/sops-nix "sops-nix"
[agenix]: https://github.com/ryantm/agenix "agenix"
