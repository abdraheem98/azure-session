# Managed Identity

## Types
- System Assigned
- User Assigned

## Portal Lab
- Enable System Assigned Identity on a VM
- Assign Reader role
- Validate identity in Enterprise Applications

## CLI

```bash
az vm identity assign -g <rg> -n <vm>
az vm identity show -g <rg> -n <vm>
```
