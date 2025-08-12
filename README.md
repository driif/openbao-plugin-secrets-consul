# OpenBao Plugin: Consul Secrets Backend

This is a standalone backend plugin for use with
[OpenBao](https://www.github.com/openbao/openbao). This plugin generates Consul
ACL tokens dynamically based on pre-configured Consul ACL policies and roles.

## Quick Links
- Main Project Github: https://www.github.com/openbao/openbao
- Plugin Repository: https://github.com/driif/openbao-plugin-secrets-consul

## Getting Started

This is an OpenBao plugin and is meant to work with OpenBao. This guide assumes
you have already installed OpenBao and have a basic understanding of how OpenBao
works.

To learn specifically about how plugins work, see documentation on [OpenBao
plugins](https://github.com/openbao/openbao/blob/development/website/content/docs/plugins/index.mdx).

## Features

- Dynamic generation of Consul ACL tokens
- Role-based token generation with configurable policies
- Automatic token revocation and cleanup
- Support for Consul Enterprise features
- Configurable token TTL and lease management

## Usage

### Configuration

First, configure the plugin with your Consul connection details:

```sh
$ bao write consul/config/access \
    address="127.0.0.1:8500" \
    token="your-consul-management-token"
```

### Creating Roles

Create roles that define the policies for generated tokens:

```sh
$ bao write consul/roles/my-role \
    consul_policies="policy1,policy2" \
    ttl="1h" \
    max_ttl="24h"
```

### Generating Tokens

Generate a Consul token for a specific role:

```sh
$ bao read consul/creds/my-role
```

## Installation & Setup

### Building the Plugin

If you wish to work on this plugin, you'll first need
[Go](https://www.golang.org) installed on your machine (Go 1.24+ recommended).

To compile a development version of this plugin, run `go build` in the repository root:

```sh
$ go build -o openbao-plugin-secrets-consul
```

This will create the plugin binary `openbao-plugin-secrets-consul` in the repository root.

### Installing the Plugin

Put the plugin binary into a location of your choice. This directory will be specified as the
[`plugin_directory`](https://github.com/openbao/openbao/blob/development/website/content/docs/configuration/index.mdx)
in the OpenBao config used to start the server.

```hcl
plugin_directory = "path/to/plugin/directory"
```

Start an OpenBao server with this config file:
```sh
$ bao server -config=path/to/config.hcl
```

### Registering the Plugin

Once the server is started, register the plugin in the OpenBao server's plugin catalog:

```sh
$ bao plugin register \
        -sha256=<expected SHA256 Hex value of the plugin binary> \
        secret \
        openbao-plugin-secrets-consul
```

Note you should generate a new sha256 checksum if you have made changes to the plugin. 
Example using `sha256sum`:

```sh
$ sha256sum openbao-plugin-secrets-consul
1642208f51c221e1d1acecbd16dcfb6ea43909a150f35fff3fc233490b722d5e  openbao-plugin-secrets-consul
```

### Enabling the Plugin

Enable the plugin backend using the secrets enable plugin command:

```sh
$ bao secrets enable -path=consul openbao-plugin-secrets-consul
```

Successfully enabled 'openbao-plugin-secrets-consul' at 'consul'!

## API Reference

### Configuration Endpoints

#### `POST /consul/config/access`
Configure connection details for Consul

**Parameters:**
- `address` (string) - Consul server address (default: "127.0.0.1:8500")
- `token` (string) - Consul management token
- `scheme` (string) - URI scheme (http/https, default: "http")
- `ca_cert` (string) - CA certificate to use when verifying Consul server certificate, must be x509 PEM encoded.
- `client_cert` (string) - Client certificate used for Consul's TLS communication, must be x509 PEM encoded and if this is set you need to also set `client_key`.
- `client_key` (string) - Client key used for Consul's TLS communication, must be x509 PEM encoded and if this is set you need to also set `client_cert`.

#### `GET /consul/config/access`
Read current configuration (sensitive values redacted)

### Role Management Endpoints

#### `POST /consul/roles/<role_name>`
Create or update a role

**Parameters:**
- `consul_policies` (string) - Comma-separated list of policies to attach to the token. Either `consul_policies` or `consul_roles` are required for Consul 1.5 and above, or just `consul_policies` if using Consul 1.4.
- `consul_roles` (string) - Comma-separated list of Consul roles to attach to the token. Either `policies` or `consul_roles` are required for Consul 1.5 and above.
- `consul_namespace` (string) - Indicates which namespace that the token will be created within. Available in Consul 1.7 and above. (default: "default")
- `node_identities` (string) - Comma-separated list of Node Identities to attach to the token. Available in Consul 1.8.1 or above.
- `partition` (string) - Indicates which admin partition that the token will be created within. Available in Consul 1.11 and above. (default: "default")
- `service_identities` (string) - Comma-separated list of Service Identities to attach to the token. Available in Consul 1.5 or above.
- `local` (bool) - Indicates that the token should not be replicated globally and instead be local to the current datacenter. Available in Consul 1.4 and above. (default: false)
- `lease` (duration) - Deprecated: Use `ttl` instead.
- `ttl` (duration) - Token time-to-live (default: 1h)
- `max_ttl` (duration) - Maximum token time-to-live (default: 24h)
- `policies` (string) - Deprecated: Use `consul_policies` instead.
- `policy` (string) - Deprecated: Policy document, base64 encoded. Required for 'client' tokens. Required for Consul pre 1.4.
- `token_type` (string) - Deprecated: Which type of token to create: 'client' or 'management'. If a 'management' token, the `policy`, `policies`, and `consul_roles` parameters are not required. (default: "client")

#### `GET /consul/roles/<role_name>`
Read role configuration

#### `DELETE /consul/roles/<role_name>`
Delete a role

#### `LIST /consul/roles`
List all roles

### Token Generation Endpoints

#### `GET /consul/creds/<role_name>`
Generate a new Consul token for the specified role

**Response:**
- `token` (string) - The generated Consul ACL token
- `accessor` (string) - The accessor of the generated Consul ACL token. Available in Consul 1.5 or above.
- `lease_id` (string) - OpenBao lease ID for token management

## Development

### Prerequisites

- Go 1.24 or higher
- Access to a Consul cluster for testing
- OpenBao binary for integration testing

### Building from Source

```sh
# Clone the repository
$ git clone https://github.com/driif/openbao-plugin-secrets-consul.git
$ cd openbao-plugin-secrets-consul

# Build the plugin
$ go build -o openbao-plugin-secrets-consul

# Or build with version information
$ go build -ldflags="-X 'github.com/driif/openbao-plugin-secrets-consul/consul.ReportedVersion=v1.0.0'" -o openbao-plugin-secrets-consul
```

### Testing

If you are developing this plugin and want to verify it is still functioning 
(and you haven't broken anything else), we recommend running the tests.

To run the tests, invoke `go test`:

```sh
$ go test ./consul/...
```

You can also filter tests like so:

```sh
$ go test ./consul/... -run=TestBackend_config_Bootstrap
```

### Integration Testing

For integration testing, you'll need a running Consul cluster. You can start one locally using Docker:

```sh
# Start Consul in development mode
$ docker run -d --name consul-dev -p 8500:8500 consul:latest

# Create a test token with appropriate permissions
$ consul acl token create -description="Test token" -policy-name="global-management"
```

Then run the integration tests:

```sh
$ CONSUL_HTTP_ADDR=127.0.0.1:8500 CONSUL_HTTP_TOKEN=your-test-token go test ./consul/... -tags=integration
```

### Code Structure

- `main.go` - Plugin entry point and server setup
- `consul/backend.go` - Backend implementation and path routing
- `consul/client.go` - Consul API client wrapper
- `consul/path_config.go` - Configuration endpoint handlers
- `consul/path_roles.go` - Role management endpoint handlers
- `consul/path_token.go` - Token generation endpoint handlers
- `consul/secret_token.go` - Token secret type and lifecycle management

## Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Make your changes
4. Add tests for your changes
5. Run the test suite (`go test ./...`)
6. Commit your changes (`git commit -am 'Add some amazing feature'`)
7. Push to the branch (`git push origin feature/amazing-feature`)
8. Open a Pull Request

## License

This project is licensed under the Mozilla Public License 2.0 - see the source files for details.

## Security

If you discover a security vulnerability, please report it to the maintainers privately before disclosing it publicly.

## Support

- File issues on the [GitHub issue tracker](https://github.com/driif/openbao-plugin-secrets-consul/issues)
- Check the [OpenBao documentation](https://github.com/openbao/openbao) for general plugin usage
- Review the [Consul ACL documentation](https://developer.hashicorp.com/consul/docs/security/acl) for policy configuration
