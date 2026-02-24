---
title: Configuring Scaleway SAML SSO with Authentik
date: 2026-02-24
draft: false
tags: ["scaleway", "authentik", "saml", "terraform", "sso"]
categories: ["devops", "infrastructure"]
---

## Introduction

In previous posts, I covered [setting up Authentik with Kubernetes and FluxCD](/2025/03/17/setting-up-authentik-with-kubernetes-and-fluxcd/), [managing Authentik with Terraform](/2025/03/18/managing-authentik-with-terraform/), and integrating it with services like [Grafana](/2025/03/19/configuring-grafana-oauth-with-authentik/), [MinIO](/2025/03/20/configuring-minio-oauth-with-authentik/), and [AWS IAM Identity Center](/2025/03/21/configuring-aws-identity-center-with-authentik-scim/). Today, I'll walk through how I configured SAML SSO for [Scaleway](https://www.scaleway.com/) using Authentik as the identity provider, all managed with Terraform.

[Scaleway](https://www.scaleway.com/) is a European cloud provider headquartered in Paris. As part of a broader move towards digital sovereignty and reducing dependency on US-based cloud providers, Scaleway offers an attractive alternative with data centers located exclusively in Europe. Integrating it with a self-hosted identity provider like Authentik fits well into this approach: keeping both infrastructure and identity management under European jurisdiction.

Unlike the AWS integration, Scaleway's SCIM provisioning is currently in early access and not yet generally available. This means users need to be pre-provisioned in the Scaleway organization manually. Despite this limitation, SAML SSO still provides a centralized authentication experience.

## Integration Overview

The integration involves:

1. **Authentik as SAML Identity Provider (IdP)**: Issues SAML assertions for authenticated users
2. **Scaleway as SAML Service Provider (SP)**: Consumes SAML assertions and grants access to the console
3. **Terraform**: Manages the Authentik-side configuration (SAML provider, application, group)
4. **Manual Scaleway setup**: Identity federation configuration in the Scaleway console, including user provisioning

Since SCIM is not available, the NameID in the SAML assertion must match an existing user in the Scaleway organization. I chose to map on **username** rather than email, as this proved more reliable for matching users across both systems.

## Scaleway Side: Setting up Identity Federation

In the Scaleway console, navigate to **IAM > Identity Providers** and create a new SAML identity provider. You'll need:

1. **SSO URL**: The Authentik SAML POST binding URL (e.g., `https://authentik.apps.example.com/application/saml/scaleway-console/sso/binding/post/`)
2. **Entity ID**: Your Authentik instance URL (e.g., `https://authentik.apps.example.com/`)
3. **Signing certificate**: Download from Authentik's SAML provider metadata and upload to Scaleway

After configuring the identity provider, Scaleway provides:

- **ACS URL**: The assertion consumer service URL (e.g., `https://api.scaleway.com/iam/v1/saml/<provider-id>/login`)
- **Entity ID**: The service provider entity ID (e.g., `https://iam.scaleway.com/saml/<provider-id>`)

You'll also need to ensure each user who should log in via SSO exists as an **IAM member** in the Scaleway organization. The organization owner alone is not sufficient - users must be explicitly added as members with matching usernames.

## Authentik Side: Terraform Configuration

With the Scaleway-side values in hand, I configured the Authentik SAML provider using Terraform:

```terraform
# Scaleway users group
resource "authentik_group" "scaleway_users" {
  name = "scaleway-users"
}

# Authentik SAML Provider for Scaleway
resource "authentik_provider_saml" "scaleway" {
  name               = "Scaleway"
  authorization_flow = data.authentik_flow.default_authorization_flow.id
  invalidation_flow  = data.authentik_flow.default_invalidation_flow.id
  acs_url            = var.scaleway_saml_acs_url
  audience           = var.scaleway_saml_entity_id
  issuer             = var.authentik_url
  signing_kp         = data.authentik_certificate_key_pair.default.id
  sp_binding         = "post"
  sign_assertion     = true
  sign_response      = true
  property_mappings  = []
  name_id_mapping    = data.authentik_property_mapping_provider_saml.username.id
}

# Authentik Application for Scaleway
resource "authentik_application" "scaleway" {
  name              = "Scaleway Console"
  slug              = "scaleway-console"
  protocol_provider = authentik_provider_saml.scaleway.id
  meta_launch_url   = "https://console.scaleway.com"
  open_in_new_tab   = true
}

# Restrict access to scaleway-users group
resource "authentik_policy_binding" "scaleway_access" {
  target = authentik_application.scaleway.uuid
  group  = authentik_group.scaleway_users.id
  order  = 0
}
```

The key settings to highlight:

1. **`issuer`**: Must be set to your Authentik instance URL (`var.authentik_url`), not the Scaleway entity ID. This is what Scaleway expects as the `<Issuer>` element in the SAML response.
2. **`sp_binding = "post"`**: Scaleway expects the SAML response to be delivered via HTTP POST.
3. **`sign_assertion = true` and `sign_response = true`**: Scaleway requires both the assertion and the response to be signed.
4. **`name_id_mapping`**: Uses the built-in username mapping rather than email, so the NameID in the SAML assertion matches the Scaleway username.
5. **`authentik_policy_binding`**: Restricts access to the Scaleway application to members of the `scaleway-users` group. Without this, any Authentik user could authenticate.

The variables are defined as:

```terraform
variable "scaleway_saml_acs_url" {
  type        = string
  description = "Scaleway SAML Assertion Consumer Service URL (from Scaleway IAM Identity Federation settings)"
}

variable "scaleway_saml_entity_id" {
  type        = string
  description = "Scaleway SAML Entity ID / Audience (from Scaleway IAM Identity Federation settings)"
}
```

And the data sources referenced:

```terraform
data "authentik_property_mapping_provider_saml" "username" {
  managed = "goauthentik.io/providers/saml/username"
}
```

## Troubleshooting

Getting SAML SSO working required solving three distinct issues. Here's what I encountered and how each was resolved.

### CSRF 403: Redirect vs POST Binding

**Symptom**: When initiating SSO from Scaleway, the AuthnRequest POST to Authentik failed with a 403 CSRF error before the login form was presented.

**Cause**: In the SP-initiated SAML flow, Scaleway sends an AuthnRequest to Authentik via HTTP POST. If the SSO URL points to the redirect binding (`/sso/binding/redirect/`), Django's CSRF protection blocks it because the redirect binding is designed for GET requests. The POST binding endpoint (`/sso/binding/post/`) is CSRF-exempt in Authentik, bypassing CSRF validation for this view.

**Fix**: Ensure the SSO URL configured in Scaleway uses the POST binding path:
```
https://authentik.apps.example.com/application/saml/scaleway-console/sso/binding/post/
```

No changes to Django's `CSRF_TRUSTED_ORIGINS` are needed — the POST binding endpoint is CSRF-exempt, so no additional configuration is needed.

### Issuer Mismatch

**Symptom**: After fixing the CSRF issue, Scaleway returned: *"response Issuer does not match the IDP metadata (expected `https://authentik.apps.example.com/`)"*.

**Cause**: The Terraform configuration initially set `issuer = var.scaleway_saml_entity_id`, which put Scaleway's own entity ID into the SAML response's `<Issuer>` element. Scaleway expects the issuer to be the **identity provider's** entity ID, not its own.

**Fix**: Change `issuer` to `var.authentik_url` (the Authentik instance URL), which matches the Entity ID configured on the Scaleway side.

### User Not Found

**Symptom**: After fixing the issuer, Scaleway returned: *"The user `user@example.com` was not found inside the organization"*.

**Cause**: The NameID mapping was set to `email`, sending `user@example.com` as the NameID. Although the user existed in the Scaleway organization with that email, the matching didn't work as expected.

**Fix**: Switch the `name_id_mapping` from the email property mapping to the username property mapping. This sends the username as the NameID, which matches the Scaleway username directly. Also ensure the user is added as an **IAM member** (not just the organization owner) in the Scaleway organization.

## Conclusion

Integrating Scaleway with Authentik via SAML SSO centralizes authentication alongside other services like AWS and Grafana. While the lack of SCIM provisioning means users must be pre-created in Scaleway's IAM, the SAML integration still provides a single sign-on experience.

The key takeaways from this integration:

- Use the **POST binding** URL for the SSO endpoint — it is CSRF-exempt in Authentik, avoiding issues with cross-origin POST requests
- The SAML `issuer` must be the **identity provider's** entity ID, not the service provider's
- Match the **NameID mapping** to however users are identified in the service provider (username in Scaleway's case)

## Resources

- [Scaleway: How to Set Up Identity Federation](https://www.scaleway.com/en/docs/iam/how-to/set-up-identity-federation/)
- [Scaleway: How to Set Up SSO with Authentik](https://www.scaleway.com/en/docs/iam/how-to/set-up-sso-with-authentik/)
- [Authentik SAML Provider Documentation](https://docs.goauthentik.io/docs/providers/saml)
- [Authentik Terraform Provider](https://registry.terraform.io/providers/goauthentik/authentik/latest/docs)
- [SAML-tracer, a Firefox extension for debugging SAML communication](https://github.com/SimpleSAMLphp/SAML-tracer)
