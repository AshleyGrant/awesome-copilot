# AzureRM Set-Type Attributes Reference

This document explains the overview and maintenance of `azurerm_set_attributes.json`.

> **Last Updated**: November 25, 2026

## Overview

`azurerm_set_attributes.json` is a definition file for attributes treated as Set-type in the AzureRM Provider.
The `analyze_plan.py` script reads this JSON to identify "false-positive diffs" in Terraform plans.

### What are Set-Type Attributes?

Terraform's Set type is a collection that **does not guarantee order**.
Therefore, when adding or removing elements, unchanged elements may appear as "changed".
This is called a "false-positive diff".

## JSON File Structure

### Basic Format

```json
{
  "resources": {
    "azurerm_resource_type": {
      "attribute_name": "key_attribute"
    }
  }
}
```

- **key_attribute**: The attribute that uniquely identifies Set elements (e.g., `name`, `id`)
- **null**: When there is no key attribute (compare entire element)

### Nested Format

When a block contains Set attributes:

```json
{
  "vpn_client_configuration": {
    "_key": null,
    "root_certificate": "name",
    "revoked_certificate": "name"
  }
}
```

- **`_key`**: The key attribute for that level's elements (`null` when the block has a single element or no key)
- **Other keys**: Definitions for nested Set attributes

### Example: azurerm_application_gateway

```json
"azurerm_application_gateway": {
  "backend_address_pool": "name",           // Simple Set (key is name)
  "http_listener": "name",
  "request_routing_rule": "name"
}
```

Only `TypeSet` blocks belong here. Application Gateway blocks such as
`url_path_map` (with `path_rule`), `rewrite_rule_set`, `frontend_ip_configuration`,
`ssl_profile`, and `trusted_root_certificate` are `TypeList` blocks. A reordering
in a list is a real change: Application Gateway evaluates path rules in the order
they are listed, so reordering overlapping path rules changes which backend serves
a request.

## Maintenance

### Adding New Attributes

1. **Check Official Documentation**
   - Search for the resource in [Terraform Registry](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs)
   - Verify the attribute is listed as "Set of ..."
   - Some resources like `azurerm_application_gateway` have Set attributes noted explicitly

2. **Check Source Code (more reliable)**
   - Search for the resource in [AzureRM Provider GitHub](https://github.com/hashicorp/terraform-provider-azurerm)
   - Confirm `Type: pluginsdk.TypeSet` in the schema definition
   - Do not add `Type: pluginsdk.TypeList` blocks: their order is part of the configuration, so an order-only change in a list is a real change
   - Identify attributes within the Set's `Schema` that can serve as `_key`

3. **Add to JSON**
   ```json
   "azurerm_new_resource": {
     "set_attribute": "key_attribute"
   }
   ```

4. **Test**
   ```bash
   # Verify with an actual plan
   python3 scripts/analyze_plan.py your_plan.json
   ```

### Identifying Key Attributes

| Common Key Attribute | Usage |
|---------------------|-------|
| `name` | Named blocks (most common) |
| `id` | Resource ID reference |
| `location` | Geographic location |
| `address` | Network address |
| `host_name` | Hostname |
| `null` | When no key exists (compare entire element) |

## Related Tools

### analyze_plan.py

Analyzes Terraform plan JSON to identify false-positive diffs.

```bash
# Basic usage
terraform show -json plan.tfplan | python3 scripts/analyze_plan.py

# Read from file
python3 scripts/analyze_plan.py plan.json

# Use custom attribute file
python3 scripts/analyze_plan.py plan.json --attributes /path/to/custom.json
```

## Supported Resources

Please refer to `azurerm_set_attributes.json` directly for currently supported resources:

```bash
# List resources
jq '.resources | keys' azurerm_set_attributes.json
```

Key resources:
- `azurerm_application_gateway` - Backend pools, listeners, rules, etc.
- `azurerm_firewall_policy_rule_collection_group` - Rule collections
- `azurerm_frontdoor` - Backend pools, routing
- `azurerm_network_security_group` - Security rules
- `azurerm_virtual_network_gateway` - IP configuration, VPN client configuration

## Notes

- Attribute behavior may differ depending on Provider/API version
- New resources and attributes need to be added as they become available
- Defining all levels of deeply nested structures improves accuracy
