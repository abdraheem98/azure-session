# Terraform RBAC

```hcl
resource "azurerm_role_assignment" "reader" {
  scope                = azurerm_resource_group.rg.id
  role_definition_name = "Reader"
  principal_id         = var.user_object_id
}
```

## Mapping

- principal_id → WHO
- role_definition_name → WHAT
- scope → WHERE

Also covered:
- Managed Identity with Terraform
- Custom Roles with Terraform
