# mTLS Implementation Guide for terraform-google-lb-http Module

## Overview

This guide explains how to use the enhanced `terraform-google-lb-http` module with mutual TLS (mTLS) authentication support. mTLS provides an additional layer of security by requiring clients to present valid certificates when connecting to your load balancer.

## What is mTLS?

Mutual TLS (mTLS) is a security protocol that requires both the server and client to authenticate each other using certificates. Unlike standard TLS where only the server presents a certificate, mTLS ensures that:

- The server verifies the client's identity before allowing access
- Both parties encrypt communications using TLS
- Only clients with valid, trusted certificates can connect

This is particularly useful for:
- API-to-API communication
- Microservices authentication
- Zero-trust network architectures
- Compliance requirements (PCI-DSS, HIPAA, etc.)

## Prerequisites

Before implementing mTLS, you need:

1. **Trust Config Resource**: A Google Cloud Trust Config that contains the CA certificates used to validate client certificates
2. **Client Certificates**: Clients must have valid certificates signed by a CA in your Trust Config
3. **Server Certificates**: Your load balancer needs SSL certificates (existing requirement)

## Module Changes

The module now includes the following new variables for mTLS support:

### Required Variables (when mTLS is enabled)

- **`enable_mtls`**: Boolean to enable/disable mTLS (default: `false`)
- **`mtls_trust_config`**: Trust config resource name for validating client certificates

### Optional Variables

- **`mtls_policy_name`**: Custom name for the server TLS policy (default: auto-generated)
- **`mtls_client_validation_mode`**: How to handle client certificate validation
  - `REJECT_INVALID` (default): Reject connections with invalid/missing client certificates
  - `ALLOW_INVALID_OR_MISSING_CLIENT_CERT`: Allow connections even without valid client certificates
- **`mtls_client_validation_trust_config`**: Alternative trust config for client validation (defaults to `mtls_trust_config`)

## Implementation Steps

### Step 1: Create a Trust Config

First, create a Trust Config in Google Cloud that contains your CA certificates:

```bash
# Create a trust config with your CA certificate
gcloud certificate-manager trust-configs create my-trust-config \
  --trust-store=trust-anchors=ca-certificate.pem \
  --location=global \
  --project=your-project-id
```

Alternatively, create it using Terraform:

```hcl
resource "google_certificate_manager_trust_config" "mtls_ca" {
  name        = "my-mtls-trust-config"
  location    = "global"
  project     = var.project_id

  trust_stores {
    trust_anchors {
      pem_certificate = file("path/to/ca-certificate.pem")
    }
  }
}
```

### Step 2: Update Your Load Balancer Configuration

Add mTLS configuration to your existing load balancer module:

```hcl
module "load_balancer_with_mtls" {
  source = "github.com/lucidworks/terraform-google-lb-http"

  name                  = "my-secure-lb"
  project               = "my-project-id"
  load_balancing_scheme = "EXTERNAL_MANAGED"

  # Standard SSL configuration (required)
  ssl                  = true
  use_ssl_certificates = true
  ssl_certificates     = [
    google_compute_ssl_certificate.my_cert.name
  ]

  # mTLS configuration
  enable_mtls                     = true
  mtls_trust_config              = google_certificate_manager_trust_config.mtls_ca.id
  mtls_client_validation_mode    = "REJECT_INVALID"

  # Optional: custom policy name
  mtls_policy_name = "my-custom-mtls-policy"

  # Your backends configuration
  backends = {
    default = {
      protocol  = "HTTP"
      port      = 80
      port_name = "http"

      health_check = {
        protocol           = "HTTP"
        port               = 80
        request_path       = "/health"
        check_interval_sec = 5
        timeout_sec        = 5
      }

      log_config = {
        enable      = true
        sample_rate = 1.0
      }

      groups = []

      iap_config = {
        enable = false
      }
    }
  }
}
```

### Step 3: Apply the Configuration

```bash
terraform init
terraform plan
terraform apply
```

## Complete Example

Here's a complete example combining all components:

```hcl
# Define local variables
locals {
  project_id   = "my-project-id"
  cluster_name = "my-cluster"
  region       = "us-central1"
}

# Create Trust Config for mTLS
resource "google_certificate_manager_trust_config" "client_ca" {
  name     = "client-ca-trust-config"
  location = "global"
  project  = local.project_id

  trust_stores {
    trust_anchors {
      pem_certificate = file("${path.module}/certificates/client-ca.pem")
    }
  }

  description = "Trust config for validating client certificates"
}

# Create or reference your server SSL certificate
resource "google_compute_ssl_certificate" "server_cert" {
  name        = "my-server-certificate"
  project     = local.project_id
  private_key = file("${path.module}/certificates/server-key.pem")
  certificate = file("${path.module}/certificates/server-cert.pem")
}

# Create load balancer with mTLS
module "mtls_load_balancer" {
  source = "github.com/lucidworks/terraform-google-lb-http"

  name                  = "mtls-enabled-lb"
  project               = local.project_id
  enable_ipv6           = false
  create_ipv6_address   = false
  http_forward          = true
  load_balancing_scheme = "EXTERNAL_MANAGED"

  # SSL Configuration
  ssl                  = true
  use_ssl_certificates = true
  ssl_certificates     = [
    google_compute_ssl_certificate.server_cert.name
  ]

  # mTLS Configuration
  enable_mtls                  = true
  mtls_trust_config           = google_certificate_manager_trust_config.client_ca.id
  mtls_client_validation_mode = "REJECT_INVALID"
  mtls_policy_name            = "my-mtls-policy"

  # Firewall configuration
  firewall_networks = [
    "${local.project_id}-${local.cluster_name}"
  ]

  # Backend services
  backends = {
    api-service = {
      description                     = "API Service Backend"
      protocol                        = "HTTP"
      port                            = 8080
      port_name                       = "http"
      timeout_sec                     = 30
      connection_draining_timeout_sec = 60
      enable_cdn                      = false
      session_affinity                = "CLIENT_IP"

      health_check = {
        protocol            = "HTTP"
        check_interval_sec  = 10
        timeout_sec         = 5
        healthy_threshold   = 2
        unhealthy_threshold = 2
        port                = 8080
        request_path        = "/healthz"
        logging             = true
      }

      log_config = {
        enable      = true
        sample_rate = 1.0
      }

      groups = []

      iap_config = {
        enable = false
      }
    }
  }
}

# Outputs
output "load_balancer_ip" {
  description = "Load balancer external IP"
  value       = module.mtls_load_balancer.external_ip
}

output "mtls_enabled" {
  description = "mTLS status"
  value       = module.mtls_load_balancer.mtls_enabled
}
```

## Testing mTLS Configuration

### Testing with curl

Once deployed, test your mTLS configuration:

```bash
# Without client certificate (should fail with REJECT_INVALID mode)
curl https://your-lb-ip.com/api

# With client certificate (should succeed)
curl --cert client-cert.pem \
     --key client-key.pem \
     --cacert ca-cert.pem \
     https://your-lb-ip.com/api
```

### Testing with openssl

```bash
# Test TLS handshake with client certificate
openssl s_client \
  -connect your-lb-ip:443 \
  -cert client-cert.pem \
  -key client-key.pem \
  -CAfile ca-cert.pem \
  -showcerts
```

## Validation Modes Explained

### REJECT_INVALID (Recommended for Production)

- **Behavior**: Rejects all connections without valid client certificates
- **Use case**: Production environments requiring strict authentication
- **Security**: Highest - only authenticated clients can connect

```hcl
mtls_client_validation_mode = "REJECT_INVALID"
```

### ALLOW_INVALID_OR_MISSING_CLIENT_CERT

- **Behavior**: Allows connections even without client certificates
- **Use case**: Migration periods, monitoring endpoints, gradual rollout
- **Security**: Lower - useful for testing but doesn't enforce mTLS

```hcl
mtls_client_validation_mode = "ALLOW_INVALID_OR_MISSING_CLIENT_CERT"
```

## Migration Strategy

When enabling mTLS on an existing load balancer:

### Phase 1: Monitoring Mode (Week 1-2)
```hcl
enable_mtls                  = true
mtls_client_validation_mode = "ALLOW_INVALID_OR_MISSING_CLIENT_CERT"
```
- Enable mTLS but allow connections without certificates
- Monitor logs to identify clients that need certificates
- Distribute client certificates to all services

### Phase 2: Enforcement (Week 3+)
```hcl
enable_mtls                  = true
mtls_client_validation_mode = "REJECT_INVALID"
```
- Switch to strict validation
- Only clients with valid certificates can connect
- Monitor for any rejected connections

## Troubleshooting

### Common Issues

#### 1. "certificate signed by unknown authority"
**Cause**: Client's CA is not in the Trust Config
**Solution**: Add the client's CA certificate to your Trust Config

#### 2. "handshake failure"
**Cause**: Client not presenting certificate or using wrong certificate
**Solution**: Ensure client is configured to send certificate during TLS handshake

#### 3. "Policy not found" error during apply
**Cause**: Server TLS Policy resource timing issue
**Solution**: The module includes proper `depends_on` - try `terraform apply` again

#### 4. Connections rejected after enabling mTLS
**Cause**: Clients not configured with certificates
**Solution**: Start with `ALLOW_INVALID_OR_MISSING_CLIENT_CERT` mode during migration

### Debug Steps

1. **Check Trust Config**:
   ```bash
   gcloud certificate-manager trust-configs describe my-trust-config \
     --location=global --project=your-project-id
   ```

2. **Verify Server TLS Policy**:
   ```bash
   gcloud compute ssl-policies list --project=your-project-id
   ```

3. **Check Load Balancer Logs**:
   ```bash
   gcloud logging read "resource.type=http_load_balancer" \
     --project=your-project-id --limit=50
   ```

4. **Verify Client Certificate Chain**:
   ```bash
   openssl verify -CAfile ca-cert.pem client-cert.pem
   ```

## Security Best Practices

1. **Certificate Rotation**: Regularly rotate client and server certificates
2. **Least Privilege**: Only add necessary CAs to Trust Config
3. **Monitoring**: Enable logging and monitor for rejected connections
4. **Certificate Expiry**: Set up alerts for certificate expiration
5. **Revocation**: Implement CRL or OCSP for certificate revocation checking

## Module Outputs

The module provides the following outputs related to mTLS:

- **`mtls_enabled`**: Boolean indicating if mTLS is enabled
- **`mtls_policy`**: The server TLS policy resource (if enabled)
- **`https_proxy`**: The HTTPS proxy (includes mTLS policy when enabled)

## Additional Resources

- [Google Cloud mTLS Documentation](https://cloud.google.com/load-balancing/docs/https/setting-up-mtls)
- [Certificate Manager Trust Configs](https://cloud.google.com/certificate-manager/docs/trust-configs)
- [TLS Best Practices](https://cloud.google.com/load-balancing/docs/ssl-policies-concepts)

## Support

For issues or questions:
1. Check the troubleshooting section above
2. Review Google Cloud documentation
3. Open an issue in the module repository
