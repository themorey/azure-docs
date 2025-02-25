---
title: "PowerShell script: Set allowed VM sizes"
description: This article includes a sample PowerShell script that sets allowed virtual machine (VM) sizes in Azure Lab Services.
ms.devlang: azurecli
ms.topic: sample
ms.date: 08/11/2020
---

# Use PowerShell to set allowed VM sizes in Azure Lab Services

This sample PowerShell script sets allowed virtual machine (VM) sizes in Azure Lab Services.

[!INCLUDE [updated-for-az](../../../includes/updated-for-az.md)]

[!INCLUDE [sample-powershell-install](../../../includes/sample-powershell-install-no-ssh-az.md)]

## Prerequisites
* **A lab**. The script requires you to have an existing lab. 

## Sample script

[!code-powershell[main](../../../powershell_scripts/devtest-lab/set-allowed-vm-sizes-in-lab/set-allowed-vm-sizes-in-lab.ps1 "Add external user to a lab")]

## Script explanation

This script uses the following commands: 

| Command | Notes |
|---|---|
| Find-AzResource | Searches for resources based on specified parameters. |
| [Get-AzResource](/powershell/module/az.resources/get-azresource) | Gets resources. |
| [Set-AzResource](/powershell/module/az.resources/set-azresource) | Modifies a resource. |
| [New-AzResource](/powershell/module/az.resources/new-azresource) | Create a resource. |

## Next steps

For more information on the Azure PowerShell, see [Azure PowerShell documentation](/powershell/).

Additional Azure Lab Services PowerShell script samples can be found in the [Azure Lab Services PowerShell samples](../samples-powershell.md).
