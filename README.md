vSTIG
=====

Automated Tool for security policy application and auditing for VMware ESXi 5.x

Overview
=====

vSTIG is a scalable plugin-based PowerCLI application intended for setting and auditing of security policies for
VMware vSphere 5.X environments.

In addition to the basic engine, aka 'vstig.ps1', several directories are available where droppable plugins and
libraries can be placed to extend the functionality of the system.

The initial focus is the USA DISA policy as dictated at http://iase.disa.mil/stigs/os/virtualization/esx.html
but any custom policy set can be deployed.

File Structure
======

The core 'vstig.ps1' script can live anywhere and is accompanied by several directories:

```
./etc/vstig-settings.ini (the core settings file)

./lib/*.ps1 (where the libraries live)

./logs/*.log (where the logs are written)

./plugins/*.ps1 (where plugins are dropped.
```

Supported Plugin Types
=====

VM - Plugins for individual VM settings.

ESXI - Plugins for ESXi settings.

VMNET - Plugins for ESXi standard and distributed vSwitch Settings

VC - Plugins for vCenter Server

VUM - Plugins for VMware Update Manager


Creating a policy
=====

While all policies share a common set of functions (in ./lib) individual checks for your desired policy must be created
through a separate plugin set.  The DISA policy can be used as a guide.

Simply create a new folder in plugins named after your policy, for example:

```  mkdir v:\vSTIG\plugins\mypolicy```

and then create the required subfolders, for example:
```
  mkdir v:\vSTIG\plugins\mypolicy\VM
  
  mkdir v:\vSTIG\plugins\mypolicy\ESXI
  
  mkdir v:\vSTIG\plugins\mypolicy\VMNET
  
  mkdir v:\vSTIG\plugins\mypolicy\VC
  
  mkdir v:\vSTIG\plugins\mypolicy\VUM
```  
  
Then add plugins to these folders as described in the following section.


Creating a Plugin
======

Create a plugin in one of the directories described above using the following header, substituting the
required variables and plugin funtion:

```
$StigSetting="RemoteDisplay.maxConnections"
$stigid="ESXI5-VM-000039"
$description='The system must limit sharing of console connections'
$LocalMode="ENFORCE"
$enabled=$true
$stigValue="1"

plugin-header $stigSetting $stigid $description $LocalMode
...
```

For manual checks the '$stigValue' should be manual:

```
$StigSetting="The system must disconnect unauthorized floppy devices (manual check)"
$stigid="ESXI5-VM-000034"
$description='The system must disconnect unauthorized floppy devices.'
$LocalMode="ENFORCE"
$enabled=$true
$stigValue="manual"

plugin-header $stigSetting $stigid $description $LocalMode
```

Your plugin must also contain logic on how to check the setting.  While this can be coded directly in the plugin it is
advisable to write a reusable function (see below chapter):

```
$StigSetting="RemoteDisplay.maxConnections"
$stigid="ESXI5-VM-000039"
$description='The system must limit sharing of console connections'
$LocalMode="ENFORCE"
$enabled=$true
$stigValue="1"

plugin-header $stigSetting $stigid $description $LocalMode

Run-VMplugin $stigSetting $enabled $localMode $stigValue

exit 0
```

In the above example, the plugin calls the 'Run-VMplugin' to audit or enforce the setting.


Creating a function
=====

Unlike plugins, all functions are common to all policies.  A single ./lib directory exists to better leverage login across all policies.

A function is simply a snippet of PowerShell code that can be called by plugins

As you would create a function or cmdlet in PowerShell, simply drop in a file containing function logic, for example:

```
function Run-VMplugin ($stigSetting, $enabled, $LocalMode, $stigValue){

  if (!(checkEnabled $enabled)) {
		warn "check $stigSetting is not enabled"
		return
	}

	if ($AllowOverrides) {
		$Mode=$LocalMode
	}

	foreach ($vm in $vms) {
		if ($stigValue.toString() -eq "manual") {
			warn "$vm - check must be run manually"
			continue
		}
		
		$setting=($vm | Get-AdvancedSetting -Name $stigSetting)
 		if ($setting) {
			if ($setting.value -ne $stigValue) {
				if ($Mode -eq "ENFORCE") {
					$vm.name
					Print-Action $vm "setting is not compliant, setting to $stigValue"
					$setting | Set-AdvancedSetting -value $stigValue -Confirm:$false | Out-Null
				} else {
					Print-Warn "$vm - NON-COMPLIANCE - but mode is $LocalMode" -foregroundcolor yellow
				}
			} else {
				Print-Report green $setting.Entity "is compliant"
			}
		} else {
			if ($Mode -eq "ENFORCE") {
				Print-Action $vm "setting does not exist, setting to $stigValue"
				$vm | New-AdvancedSetting -Name $stigSetting -value $stigValue -Confirm:$false | Out-Null
			} else {
				warn "$vm - Setting doesn't exist but mode is $LocalMode"
			}
		}
	}

}
```

You can create new functions in this manner to extend your plugins without breaking plugins that were written before.
