![alt text](https://raw.githubusercontent.com/Scalingo/apache-buildpack/1c1e05f53f1d611e0dd2141bd8a49a7fa6cebcbe/scalingo_apache.svg?raw=true)

# Apache Buildpack

This buildpack installs the Apache2 web server using the `apt-buildpack`.

## Configuration

If an `apache.conf.erb` is present at the root of your project, it will be
rendered (using ERB template system) and included in the default configuration.

## Apache modules

You can load specific Apache modules by adding a `.apache-mods` file at the
root of your project. The buildpack automatically installs the module(s) if
necessary and updates the Apache configuration to load them accordingly.

The file must include one module per line.

The list of modules available, with a short description of each module, is
available at the following locations, depending on the version of the stack
used:
* Stack `scalingo-20`: https://packages.ubuntu.com/search?suite=focal&searchon=names&keywords=libapache2-mod
* Stack `scalingo-18`: https://packages.ubuntu.com/search?suite=bionic&searchon=names&keywords=libapache2-mod

Please note you only need to pass the module name inside the `.apache-mods`
file, dropping the `libapache2-mod-` part.

Example:

```
auth-openidc
auth-mellon
```

> **Warning**\
> If the module you try to load isn't a default module (embedded with Apache),
and if it doesn't exist (see lists of available modules above), the buildpack
will fail.


## Environment tweaks

* `APACHE_LOG_LEVEL`: (default: `info`) Define the apache log level among the
  following:
  * `emerg`	System is unusable
  * `alert`	Action must be taken immediately
  * `crit`	Critical conditions
  * `error`	Error conditions
  * `warn`	Warning conditions
  * `notice`	Normal, but significant conditions
  * `info`	Informational messages
  * `debug`	Debugging messages
  * `trace1` – `trace8` Trace messages with gradually increasing levels of detail

## The `apache.conf.erb` file

The `apache.conf.erb` file can be used to include specific configuration for
your environment. It should be placed at the root of your application
repository.

You may want to include variables as Ruby code in this file, which will
match environment variables from your container.

For example, you can create a variable `MY_VAR` in your container and use it in
the `apache.conf.erb` file with `<%= ENV["MY_VAR"] %>`. It will be matched to
the corresponding environment variable and replaced with its value before
starting Apache.

If no `apache.conf.erb` file is found, Apache will work as a simple HTTP server
and will serve the content of the `www` directory present on your application
repository.

## Example setting up a basic OpenID-Connect Relying Party

To configure Apache as an OpenID-Connect Relying Party (RP), you can create an
`apache.conf.erb` with the following code:

```
OIDCProviderMetadataURL <issuer>/.well-known/openid-configuration
OIDCClientID <%= ENV["OIDC_CLIENT_ID"] %>
OIDCClientSecret <%= ENV["OIDC_CLIENT_SECRET"] %>

OIDCRedirectURI https://<hostname>/secure/redirect_uri
OIDCCryptoPassphrase <%= ENV["OIDC_CRYPTO_PASSPHRASE"] %>

<Location /secure>
   AuthType openid-connect
   Require valid-user
</Location>
```
And create the following environment variables in your container:
* `OIDC_CLIENT_ID=<client_id_value>`
* `OIDC_CLIENT_SECRET=<client_secret_value>`
* `OIDC_CRYPTO_PASSPHRASE=<crypto_passphrase_value>`

## Mellon configuration

In order for `auth-mellon` to work, mellon needs to have a key, certificate and
metadata file available.

You can generate the files accordingly:
* Run the `tools/mellon_create_metadata.sh` script, passing it the `entity-id`
  and `endpoint-path` as parameters ;
* 3 files will be created. For each file, run the command:
  `base64 --wrap 0 <file_name>` ;
* Create the environment variables `MELLON_SP_KEY`, `MELLON_SP_CERT` and
  `MELLON_SP_METADATA` and paste the corresponding value returned by the above
  command.

You will also need to run the `base64 --wrap 0 <file_name>` against the
Identity Provider metadata file and copy-paste the returned value into a
`MELLON_IDP_METADATA` environment variable.

Sometimes, the `MELLON_IDP_METADATA` variable will exceed the maximum allowed
size of 8192 characters. In that case, you can:
* Put the **mellon_idp_metadata.xml** file inside a password-protected
  **mellon_idp_metadata.zip** file (**NOTE: Please respect the files names!**)
* Add a `MELLON_IDP_PASSPHRASE` environment variable containing the **base64**
  value of the password needed to unzip the file


You can then set these files in your Mellon configuration, as such:
```
[...]
MellonSPMetadataFile <%= ENV["HOME"] %>/vendor/apache2/mellon/mellon.xml
MellonSPPrivateKeyFile <%= ENV["HOME"] %>/vendor/apache2/mellon/mellon.key
MellonSPCertFile <%= ENV["HOME"] %>/vendor/apache2/mellon/mellon.cert
MellonIdPMetadataFile <%= ENV["HOME"] %>/vendor/apache2/mellon/mellon_idp_metadata.xml
[...]
```

## Variables

The following table lists all the variables you can set, and a description of
each:

| Var Name                 | Module      | Default | Description                                                                                                                     |
| ------------------------ | :---------: | :-----: | ------------------------------------------------------------------------------------------------------------------------------- |
| APACHE_LOG_LEVEL         | General     | info    | Sets the Apache log level.                                                                                                      |
| APACHE_WORKER_SIZE       | General     | 10      | Sets the average Apache process size, in Mb. Used to calculate ServerLimit and MaxRequestWorkers.                               |
| APACHE_THREADS_PER_CHILD | General     | 25      | Sets the number of threads per child. Used to calculate MaxRequestWorkers.                                                      |
| MELLON_SP_KEY            | auth-mellon | none    | Used to set the Mellon Service Provider key. Should be the key converted to base64.                                             |
| MELLON_SP_CERT           | auth-mellon | none    | Used to set the Mellon Service Provider certificate. Should be the certificate converted to base64.                             |
| MELLON_SP_METADATA       | auth-mellon | none    | Used to set the Mellon Service Provider metadata xml file. Should be the xml file converted to base64.                          |
| MELLON_IDP_METADATA      | auth-mellon | none    | Used to set the Mellon Identity Provider metadata xml file. Should be the xml file converted to base64.                         |
| MELLON_IDP_PASSPHRASE    | auth-mellon | none    | Used to set the passphrase to decrypt Mellon Identity Provider metadata xml file. Should be the passphrase converted to base64. |
| YOUR_KEY                 | any         | none    | You can set your own variables inside the apache.conf.erb file. Use string `<%= ENV["YOUR_KEY"] %>` inside apache.conf.erb.     |
