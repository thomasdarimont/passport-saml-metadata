# passport-saml-metadata

Utilities for reading configuration from SAML 2.0 Metadata XML files, such as those generated by Active Directory Federation Services (ADFS).

## Installation

```
npm install passport-saml-metadata
```

## Usage Example

```javascript
const os = require('os');
const fileCache = require('file-system-cache').default;
const { fetch, toPassportConfig, claimsToCamelCase } = require('passport-saml-metadata');
const SamlStrategy = require('passport-wsfed-saml2').Strategy;

const backupStore = fileCache({ basePath: os.tmpdir() });
const url = 'https://adfs.company.com/federationMetadata/2007-06/FederationMetadata.xml';

fetch({ url, backupStore })
  .then((reader) => {
    const config = toPassportConfig(reader);
    config.realm = 'urn:nodejs:passport-saml-metadata-example-app';
    config.protocol = 'saml2';

    passport.use('saml', new SamlStrategy(config, function(profile, done) {
      profile = claimsToCamelCase(profile, reader.claimSchema);
      done(null, profile);
    }));

    passport.serializeUser((user, done) => {
      done(null, user);
    });

    passport.deserializeUser((user, done) => {
      done(null, user);
    });
  });
```

See [compwright/passport-saml-example](https://github.com/compwright/passport-saml-example) for a complete reference implementation.

## API

### fetch(config = {})

When called, it will attempt to load the metadata XML from the supplied URL. If it fails due to a request timeout or other error, it will attempt to load from the `backupStore` cache.

Config:

* `url` (required) Metadata XML file URL
* `timeout` Time to wait before falling back to the `backupStore`, in ms (default = `2000`)
* `backupStore` Any persistent cache adapter object with `get(key)` and `set(key, value)` methods (default = `new Map()`)

Returns a promise which resolves, if successful, to an instance of `MetadataReader`.

### toPassportConfig(reader)

Transforms metadata extracts for use in Passport strategy configuration. The following strategies are currently supported:

* [passport-saml](http://npmjs.org/packages/passport-saml)
* [passport-wsfed-saml2](http://npmjs.org/packages/passport-wsfed-saml2)

### claimsToCamelCase(claims, claimSchema)

Translates the claim identifier URLs to human-friendly camelCase versions. Useful in Passport verifier functions.

`claimSchema` should be an object of the following format, such as from `MetadataReader.claimSchema()`:

```javascript
{
  [claimURL]: {
    name: claimUrl,
    camelCase: 'claimIdentifierInCamelCase',
    description: 'Some description'
  },
  ...
}
```

Example:

```javascript
function verifier(profile, done) {
  profile = passportSamlMetadata.claimsToCamelCase(profile, reader.claimSchema);
  done(null, profile);
}
```

### new MetadataReader(metadataXml, options = { throwExceptions: false })

Parses metadata XML and extracts the following properties:

* `identifierFormat` (e.g. `urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress`)
* `identityProviderUrl` (e.g. https://adfs.server.url/adfs/ls/)
* `logoutUrl` (e.g. https://adfs.server.url/adfs/ls/)
* `signingCert`
* `encryptionCert`
* `claimSchema` - an object hash of claim identifiers that may be provided in the SAML assertion

### metadata(app)(config = {})

Returns a function which sets up an Express application route to generate the metadata XML file for your application at /FederationMetadata/2007-06/FederationMetadata.xml. ADFS servers may import the resulting file to set up the relying party trust.

Config:

* `issuer` (required) The unique application identifier, used to name the relying party trust; may be a URN or URL
* `callbackUrl` (required) The absolute URL to redirect back to with the SAML assertion after logging in, usually https://hostname[:port]/login/callback
* `logoutCallbackUrl` The absolute URL to redirect back to with the SAML assertion after logging out, usually https://hostname[:port]/logout

See [compwright/passport-saml-example](https://github.com/compwright/passport-saml-example) for a usage example.
