Managed Certificates
====================

This role prepares the target system for external management of X.509 certificates.
Certificates, keys and certificate chains can be deployed to the target system by simply uploading them via SFTP as an unprivileged user.
The system itself handles basic validation of the uploaded files, moving them to the correct location and restarting services, so they load the new files.
Some configuration is required for this to work correctly (see [Role Variables](#role-variables) below).

The external certificate manager needs to fulfill the following requirements:
* All files must be uploaded directly into the upload directory. Subdirectories are not supported.
* All certificates must be in PEM format and have the file extension `.pem`.
* The host's certificate must be in a file with no other certificates.
* The certificate issuer chain must be in a file with only the issuer chain certificate(s) in the correct order.
* The certificate's private key must be in a separate file with the extension `.key`. Except for the extension, the file name must be identical to the host's certificate file name.
* The upload of all required files needs to be completed within a configurable time interval (see `managed_certificates_deployment_delay`) after the first upload started.

Further notes:
* There are no other requirements on uploaded file names. It is not necessary to use the same file names on certificate renewal.
* On certificate renewal, issuer certificates and private key do not need to be uploaded, if the new certificate matches the files that are already present on the system. If they do not match, the uploaded certificate is discarded.
* Uploaded files are renamed to conform to a consistent scheme. File names are
    * `$cn.pem` - the host's certificate
    * `$cn.key` - the private key of the host's certificate
    * `$cn_chain.pem` - the issuer certificate chain (if any)
    * `$cn_fullchain.pem` - the bundled host certificate and issuer chain
* All other uploaded files are deleted in the process.

Refer to https://github.com/s-hamann/firecracker-tools for an implementation of an external certificate manager that works with this role.

Requirements
------------

This role requires Ansible 2.10 but has no further requirements on the controller.

Role Variables
--------------

* `managed_certificates_deployment_user`  
  The name of the user account that uploads new certificates to the system.
  This user may only access the target system via SFTP and may only write to the (usually empty) upload directory and read only the `authorized_keys` file.
  Default is `deploy-cert`.
* `managed_certificates_deployment_user_extra_groups`  
  A list of groups that the `managed_certificates_deployment_user` is added to.
  All groups need to exist on the target system; this role does not create them.
  Empty by default.
* `managed_certificates_deployment_user_pubkey`  
  A list of SSH public keys used by `managed_certificates_deployment_user` to authenticate to the target system.
  May be given as the public key itself or as a path on the Ansible controller to the public key file.
  Optional.
* `managed_certificates_deployment_upload_directory`  
  The directory where externally triggered certificate uploads are expected.
  The parent directory of this directory is made the home directory of `managed_certificates_deployment_user` and the root directory for SFTP access by this user.
  If the target system does not use systemd, this path must not contain any whitespace characters.
  Default is `/var/lib/deploy-cert/upload/`.
* `managed_certificates_deployment_log_facility`  
  Status messages during the installation of certificates are logged to this syslog facility.
  Default is `local0`.
* `managed_certificates_deployment_delay`  
  Installing the uploaded files starts this many seconds after the first upload started.
  All required files MUST be uploaded in this time frame.
  Larger values delay certificate installation, smaller values may cause the deployment to fail.
  Default is `30`.
* `managed_certificates_server_group`  
  The name of a system group that is allowed read access to the certificates and private keys.
  Services that use the certificates should be added to this group.
  Default is `tls-server`.
* `managed_certificates_server_certificate_directory`  
  A directory where the server certificates and private keys are stored.
  Their file names are based on the certificate's "common name" attribute, if present, and the first "subject alternative name" otherwise.
  Default is `/etc/ssl/local_certs/`.
* `managed_certificates_server_certificate_consumers`  
  A list of services that need to be restarted when a new server certificate is deployed.
  Optional.
* `managed_certificates_client_group`  
  The name of a system group that is allowed read access to the certificates and private keys.
  Services that use the certificates should be added to this group.
  Default is `tls-client`.
* `managed_certificates_client_certificate_directory`  
  A directory where the client certificates and private keys are stored.
  Their file names are based on the certificate's "common name" attribute, if present, and the first "subject alternative name" otherwise.
  Default is `/etc/ssl/client_certs/`.
* `managed_certificates_client_certificate_consumers`  
  A list of services that need to be restarted when a new client certificate is deployed.
  Optional.
* `managed_certificates_inaccessible_paths`  
  If the target system uses systemd, this option takes a list of paths, that should not be accessible at all during the certificate installation process.
  Regardless of this option, home directories are made inaccessible.
  Optional.

Dependencies
------------

This role assumes that OpenSSH runs on the target system and assumes that the OpenSSH configuration includes files from `/etc/ssh/sshd_config.d/`.
This should be set up by some other role.

Example Configuration
---------------------

The following is a short example for some of the configuration options this role provides:

```yaml
managed_certificates_deployment_user_pubkey: 'ssh-ed25519 AAAA...'
managed_certificates_deployment_user_extra_groups:
    - ssh-users
managed_certificates_server_certificate_consumers:
    - postfix
```

License
-------

MIT
