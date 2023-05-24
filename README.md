# PureScript Registry

[![Import Packages](https://github.com/purescript/registry-preview/actions/workflows/importer.yml/badge.svg)](https://github.com/purescript/registry-preview/actions/workflows/importer.yml)
[![Registry API](https://github.com/purescript/registry-preview/actions/workflows/api.yml/badge.svg)](https://github.com/purescript/registry-preview/actions/workflows/api.yml)

The PureScript Registry stores PureScript packages (and metadata about them) and provides an API for registering, updating, transferring, and unpublishing packages. Most PureScript users will only interact with the registry via package managers.

> **Warning**
> As of September 2022, no package manager officially supports the registry. To register your package, please open an issue using the [Publish](#publish-a-package) workflow. Once registered, your package will be added to the package sets automatically (if it compiles with the current package set), its docs will be uploaded to Pursuit, and any SemVer tag you publish on the repository in the future will automatically update your package in the registry.

## Registry Overview

This repository provides three things:

1. Metadata for every package version in the registry, stored in the `metadata` directory
2. Package sets, which are updated daily and stored in the `package-sets` directory
3. An API for registering, updating, transferring, and unpublishing packages, as well as for making manual package set updates

The PureScript Registry as a whole consists of this repository plus a few more resources:

1. The [Registry Index](https://github.com/purescript/registry-index) stores the manifest files for every package version in the registry. Package managers use the registry index to rapidly determine relationships among packages at specific versions.
2. The [Registry Storage](https://packages.registry.purescript.org) stores tarballs of packages that have been uploaded to the registry. Package managers download dependencies from the registry storage.
3. The [Registry Dev](https://github.com/purescript/registry-dev) repository is used to develop the registry. The registry-dev repository is also a library, which package managers can use to easily integrate with the registry by reusing its types and functions.

## Using the Registry API

Most PureScript users will only use the Registry API via their package manager. However, it is also possible to interact with it directly via GitHub issues in this repository. The guide below separates _Operations_ (API calls anyone can make) from _Authenticated Operations_ (API calls that can only be made by package owners or registry trustees).

When you submit an issue with the correct payload, this repository's CI will execute the API. Any errors will be reported as a comment on the issue you opened; you can call the API again by commenting on the same issue you opened with another valid payload. If the operation completes successfully then that will be reported to you in a comment and the registry will close the issue.

### Operations

The Registry API allows anyone to publish a package version or upgrade the package set. To add or update a package, your package must contain a `bower.json` file, `spago.dhall` file, or [`purs.json` manifest file](https://github.com/purescript/registry-dev/blob/0bad5ddded9ca7d8f3e461588aa92a05df9125b1/src/Registry/Schema.purs#L21-L30). Eventually, the registry will no longer accept packages that do not have a registry manifest file, but for now legacy manifests are still accepted. For an example of the new manifests, see the [manifests for the `prelude` package](https://github.com/purescript/registry-index/blob/main/pr/el/prelude).

#### Publish a Package

To publish a [new package](https://github.com/purescript/registry/issues/new?title=Add+my-library&template=publish_new.md) or to [publish a new version of an existing package](https://github.com/purescript/registry/issues/new?title=Update+my-library&template=publish_update.md), open a new issue with a comment consisting of package metadata in the JSON format shown below.

> **Note**
> If your package is in the registry but it is missing documentation on Pursuit, you can re-publish an existing version to re-generate documentation.

```jsonc
// Please note that jsonc is not supported, only used here for documentation.
// Remove comments and comma-dangles below before submitting, or use the issue
// templates linked above.
{
  // These three fields are always required.
  "name": "safe-coerce",
  "ref": "v12.0.0",
  "compiler": "0.15.4",
  
  // You must specify a GitHub location if this package has never been published
  // to the registry before. Otherwise, this field is optional.
  "location": {
    "githubOwner": "purescript",
    "githubRepo": "purescript-safe-coerce"
  },
  
  // The registry will automatically solve your dependencies, so this field is
  // optional. However, you can also provide specific dependency versions to use
  // when compiling your package.
  "resolutions": { "unsafe-coerce": "6.0.0" },
}
```

The registry will fetch the repository at the provided ref and attempt to register the package, compile it with the provided compiler version, upload the documentation to Pursuit, and add the package to the day's package set batch.

#### Modify the Package Set

Package sets are released once per day automatically. However, they sometimes need manual intervention (for example, to update a set of packages all together). The registry provides a batch package set update API for this purpose. You can add new packages or update package versions using the JSON specified below. The registry will update the package set to the indicated versions and recompile it. If successful, it will release a new package set with your changes.

Only Registry Trustees are able to remove packages (by setting their value to `null` in the payload) or upgrade the compiler version (by setting the `compiler` field). If you are not a Trustee, then you should just set the `packages` field to the new desired versions for the packages you wish to update.

Example payload:

```jsonc
{
  "packages": {
    "aff": "7.1.0",
    "argonaut": "9.1.0"
  }
}
```

### Authenticated Operations

The Registry API allows package owners or Registry Trustees to transfer or unpublish packages. A "package owner" is an [Owner](https://github.com/purescript/registry/blob/24db591b992b99730ea23211d746bc4c5fe295d7/src/Registry/Schema.purs#L55-L59) listed in the package manifest; if you wish to take an authenticated action for a package then you must first publish a version in which your SSH public key is listed in the `owners` field. For example:

```jsonc
// Your public key will list these three fields in the format <keytype> <public> <email>
{
  "email": "owner@email.org", // this does not need to be a valid email address, but it must match the key
  "keytype": "ssh-ed25519",
  "public": "AAAAC3NzaC1lZDI1NTE5AAAAIK0wmN/Cr3JXqmLW7u+g9pTh+wyqDHpSAQEIg9p"
}
```

If your key is listed in the package owners, then you can transfer or unpublish your package. To do so, create the JSON payload for a `Transfer` or `Unpublish` operation, and then sign the payload with your SSH key and submit the result in the JSON format below. Let's walk through a short example of transferring a package.

First, we produce the payload for the operation we wish to take. For a transfer, that might be:

```json
{
  "name": "my-package",
  "newLocation": {
    "githubOwner": "new-owner",
    "githubRepo": "purescript-my-package"
  }
}
```

Next, place the JSON for your authenticated operation in a file. Then, you can sign the file with your SSH key using this command:

```console
ssh-keygen -Y sign -f ~/.ssh/id_ed25519 -n file payload_file
```

The signature is written to a new file called payload_file.sig, which looks like this:

```
-----BEGIN SSH SIGNATURE-----
U1NIU0lHAAAAAQAAADMAAAALc3NoLWVkMjU1MTkAAAAg2rirQQddpzEzOZwbtM0LUMmlLG
krl2EkDq4CVn/Hw7sAAAAEZmlsZQAAAAAAAAAGc2hhNTEyAAAAUwAAAAtzc2gtZWQyNTUx
OQAAAEDyjWPjmOdG8HJ8gh1CbM8WDDWoGfm+TTd8Qa8eua9Bt5Cc+43S24i/JqVWmk98qV
YXoQmOYL4bY8t/q7cSNeMH
-----END SSH SIGNATURE-----
```

Alternately, if you use `-` as the filename, then the payload will be read from stdin and the signature will be written to stdout. Now, we can produce our authenticated operation. Here's the JSON spec for an authenticated operation:

```jsonc
{
  // The JSON representation of an operation, such as a Transfer or Unpublish
  "payload":
    {
      "name": "my-package",
      "newLocation": {
        "githubOwner": "new-owner",
        "githubRepo": "purescript-my-package"
      }
    },
  // The SSH signature produced with ssh-keygen
  "signature": [
    "-----BEGIN SSH SIGNATURE-----",
    "U1NIU0lHAAAAAQAAADMAAAALc3NoLWVkMjU1MTkAAAAg2rirQQddpzEzOZwbtM0LUMmlLG"
    "krl2EkDq4CVn/Hw7sAAAAEZmlsZQAAAAAAAAAGc2hhNTEyAAAAUwAAAAtzc2gtZWQyNTUx",
    "OQAAAEDyjWPjmOdG8HJ8gh1CbM8WDDWoGfm+TTd8Qa8eua9Bt5Cc+43S24i/JqVWmk98qV",
    "YXoQmOYL4bY8t/q7cSNeMH",
    "-----END SSH SIGNATURE-----"
  ],
  // The email associated with the public SSH key
  "email": "owner@email.com"
}
```

You can now submit this payload to the registry to process the authenticated operation.

#### Transfer a Package

To transfer a package, authenticate a payload of the JSON spec below.

Example payload:

```jsonc
{
  // The name of the package to transfer
  "name": "math",
  // The GitHub location to transfer the package to. This location
  // cannot already be in use in the registry.
  "newLocation": {
    "githubOwner": "purescript-deprecated",
    "githubRepo": "purescript-math"
  }
}
```

#### Unpublish a Package Version

Packages versions can be unpublished for a short time after being published. To unpublish a recently-published package version, create an authenticated payload following the spec below.

Example payload:

```jsonc
{
  // The package to unpublish
  "name": "halogen-hooks",
  // The version of the package to unpublish
  "version": "2.0.0",
  // A short description of why the package was unpublished
  "reason": "Accidentally committed key."
}
```
