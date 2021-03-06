# PKI & CONJUR
The PKI service uses conjur as it's persistence and authorization layer. 
Meaning whenever a new resource is created, read, updated or deleted that a corresponding api call to conjur will occur.


The PKI service will configure a specific policy branch with the policies below. This default branch is `pki` 
Below are all of the resources generated by the PKI service initialization:
```yaml
- !webservice
- !variable ca/cert
- !variable ca/key
- !variable ca/cert-chain
- !variable crl

# higher level groups
- !group admin
- !group audit
- &admins
  - !group templates-admin
  - !group certificates-admin
  - !group purge-admin
  - !group ca-admin

# endpoint permission groups
- &templates
  - !group list-templates
  - !group read-templates
  - !group create-templates
  - !group manage-templates
  - !group delete-templates
- &certificates
  - !group list-certificates
  - !group read-certificates
  - !group create-certificates
  - !group sign-certificates
  - !group revoke-certificates
- &purge
  - !group purge
  - !group purge-crl
- &ca
  - !group set-ca-chain
  - !group set-ca-signing-key
  - !group set-ca-signing-cert
  - !group generate-ca
```
As you see above, we are creating groups that have the ability to perform actions on large subsets of certificates or templates like `!group read-templates`. Whenever a new template resource is created, the pki service will make sure the corresponding privilege is set to the `read-templates` group.



Also to mention that some of the groups above will not have privileges applied to them dynamically. Instead the privilege remains static such as `!group set-ca-chain` or `!group purge` where the groups are just a one to one mapping between role and privilege. Below are the example of these "static" pki service groups.
```yaml
- !permit
  role: !group list-templates
  resource: !webservice
  privileges:
  - list-templates

- !permit
  role: !group list-certificates
  resource: !webservice
  privileges:
  - list-certificates

- !permit
  role: !group purge
  resource: !webservice
  privileges:
  - purge

- !permit
  role: !group purge-crl
  resource: !webservice
  privileges:
  - purge-crl

- !permit
  role: !group set-ca-chain
  resource: !webservice
  privileges:
  - set-ca-chain

- !permit
  role: !group set-ca-signing-key
  resource: !webservice
  privileges:
  - set-ca-signing-key

- !permit
  role: !group set-ca-signing-cert
  resource: !webservice
  privileges:
  - set-ca-signing-cert

- !permit
  role: !group generate-ca
  resource: !webservice
  privileges:
- generate-ca
```

Lastly are the grants, this is where the admin groups become members to a bunch of other more granular groups.
The audit is unique as it only has the ability to list and read certificates and templates
```yaml
- !grant
  roles: *templates
  member: !group templates-admin

- !grant
  roles: *certificates
  member: !group certificates-admin

- !grant
  roles: *purge
  member: !group purge-admins

- !grant
  roles: *ca
  member: !group ca-admins

- !grant
  roles: *admins
  member: !group admin

- !grant
  roles:
  - !group list-templates
  - !group list-certificates
  - !group read-templates
  - !group read-certificates
  member: !group audit
```



## Certificates
### Create
When a certificate is created in the pki service the following policy will be loaded in the `pki` policy branch.
The certificate is repersented as a `!variable` with annotations.
You will see 2 groups are created when a certificate is created `certificates/<SerialNumber>-read` and `certificates/<SerialNumber>-revoke`. These groups will have granular access to the pki webservice to `read` or `revoke` this specific certificate. The `!grant` statement at the bottom of the policy shows how granular privileges trickle up to the higher-level `read-certificates` or `revoke-certificates` groups.
```yaml
- !variable
  id: certificates/<SerialNumber>
  annotations:
    Revoked: <Revoked>
    RevocationDate: <RevocationDate>
    RevocationReasonCode: <RevocationReasonCode>
    ExpirationDate: <ExpirationDate>
    InternalState: csasa<InternalState>csasa

# groups related to the privileges
- !group certificates/<SerialNumber>-read
- !group certificates/<SerialNumber>-revoke

# assign the privileges to the groups above
- !permit
  role: !group certificates/<SerialNumber>-read
  resource: !webservice pki
  privileges:
  - read-certificate-<SerialNumber>
- !permit
  role: !group certificates/<SerialNumber>-revoke
  resource: !webservice pki
  privileges:
  - revoke-certificate-<SerialNumber>

# assign global level groups the ability to read and revoke these certificates
- !grant
  role: !group certificates/<SerialNumber>-read
  member: !group read-certificates

- !grant
  role: !group certificates/<SerialNumber>-revoke
  member: !group revoke-certificates
```

### Revoke
Updating specific annotations is the only requirement when certificates are revoked. All metadata of a certificate is stored as annotations. Below is the policy that is loaded when revoking a certificate:
```yaml
- !variable
  id: "<SerialNumber>"
  annotations:
    Revoked: <Revoked>
    RevocationDate: <RevocationDate>
    RevocationReasonCode: <RevocationReasonCode>
    InternalState: csasa<InternalState>csasa
```
As you see the `ExpirationDate` is not present in the policy. This is done on purpose because when a policy PATCH is executed the provided annotations will be updated however the non-provided annotations will remain untouched.


### Delete
When a certificate is deleted. We must make sure to delete all of the groups that are associated with this certificate:
```yaml
- !delete
  records: 
  - !variable certificates/<SerialNumber>
  - !group certificates/<SerialNumber>-read
  - !group certificates/<SerialNumber>-revoke
```


## Templates
### Create
When a template is created by the PKI service all metadata is stored as a json string as the !variable's secret value. The following policy will be loaded. 3 groups are being created with the template: `read`, `manage` and `delete`:
```yaml
- !variable
  id: templates/<TemplateName>

# groups related to the privileges
- !group templates/<TemplateName>-read
- !group templates/<TemplateName>-manage
- !group templates/<TemplateName>-delete

# assign the privileges to the groups above
- !permit
  role: !group templates/<TemplateName>-read
  resource: !webservice pki
  privileges:
  - read-template-<TemplateName>

- !permit
  role: !group templates/<TemplateName>-manage
  resource: !webservice pki
  privileges:
  - manage-template-<TemplateName>

- !permit
  role: !group templates/<TemplateName>-delete
  resource: !webservice pki
  privileges:
  - delete-template-<TemplateName>

# grant the groups accordingly
- !grant
  role: !group templates/<TemplateName>-read
  member: !group read-templates

- !grant
  role: !group templates/<TemplateName>-manage
  member: !group manage-templates

- !grant
  role: !group templates/<TemplateName>-delete
  member: !group delete-templates
```  

## Manage
When a Template is managed no policy is loaded. Except the secret value of the templates !variable is updated to reflect the change.

## Delete
When a Template is deleted, the template !variable and its 3 corresponding !groups will be deleted. This is the policy that is loaded when a Template is deleted by the PKI service:
```yaml
- !delete
  records: 
  - !variable templates/<TemplateName>
  - !group templates/<TemplateName>-read
  - !group templates/<TemplateName>-managed
  - !group templates/<TemplateName>-delete
```
