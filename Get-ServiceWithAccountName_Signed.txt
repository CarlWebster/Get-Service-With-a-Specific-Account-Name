<#
.SYNOPSIS
	Find Services using a specific account name on Servers in Microsoft 
	Active Directory.
.DESCRIPTION
	By default, builds a list of all computers where "Server" is in the 
	OperatingSystem property unless the ComputerName or InputFile 
	parameter is used.
	
	Process each server looking for Services using a specific StartName.
	
	Builds a list of computer names, Service names, service display names and service start names.
	
	Display the matching results on the console and creates two text files, by default, 
	in the folder where the script is run.
	
	Optionally, can specify the output folder.
	
	Unless the InputFile parameter is used, needs the ActiveDirectory module.
	
	The script has been tested with PowerShell versions 2, 3, 4, 5, and 5.1.
	The script has been tested with Microsoft Windows Server 2008 R2, 2012, 
	2012 R2, and 2016 and Windows 10 Creators Update.
.PARAMETER AccountName
	Account name used for the service "log on".
	Script surrounds AccountName with "*".
	
	For example, if ctxadmin is entered, the script uses "*ctxadmin*".
	
	This allows the script to find Services where the StartName property is:
		ctxadmin@domain.com
		Domain\ctxadmin
		Domain.com\ctxadmin
	
	Side effect: Will also find Services with StartName like "This is the ctxadmin account"
	
	This is a Required parameter.
	Alias is AN
.PARAMETER LiteralAccountName
	Account name used for the service "log on".
	Unlike AccountName, the value of LiteralAccountName is used exactly as it is typed.
	
	For example, if ctxadmin is entered, the script uses "ctxadmin".
	
	This is a Required parameter.
	Alias is LAN
.PARAMETER ComputerName
	Computer name used to restrict the computer search.
	Script surrounds ComputerName with "*".
	
	For example, if "PVS" is entered, the script uses "*PVS*".
	
	This allows the script to reduce the number of servers searched.
	
	If both ComputerName and InputFile are used, ComputerName is used to filter
	the list of computer names read from InputFile.
	
	Alias is CN
.PARAMETER InputFile
	Specifies the optional input file containing computer account names to search.
	
	Computer account names can be either the NetBIOS or Fully Qualified Domain Name.
	
	ServerName and ServerName.domain.tld are both valid.
	
	If both ComputerName and InputFile are used, ComputerName is used to filter
	the list of computer names read from InputFile.
	
	The computer names contained in the input file are not validated.
	
	Using this parameter causes the script to not check for or load the ActiveDirectory module.
	
	Alias is IF
.PARAMETER OrganizationalUnit
	Restricts the retrieval of computer accounts to a specific OU tree. 
	Must be entered in Distinguished Name format. i.e. OU=XenDesktop,DC=domain,DC=tld. 
	
	The script retrieves computer accounts from the top level OU and all sub-level OUs.
	
	Alias OU
.PARAMETER Folder
	Specifies the optional output folder to save the output reports. 
.EXAMPLE
	PS C:\PSScript > .\Get-ServiceWithAccountName.ps1 -AccountName ctxadmin
	
	The script will change "ctxadmin" to "*ctxadmin*" and will seach for all 
	services using a StartName that contains "ctxadmin".
	
.EXAMPLE
	PS C:\PSScript > .\Get-ServiceWithAccountName.ps1 -AN svc_acct
	
	The script will change "svc_acct" to "*svc_acct*" and will seach for all 
	services using a StartName that contains "svc_acct".
	
.EXAMPLE
	PS C:\PSScript > .\Get-ServiceWithAccountName.ps1 -LiteralAccountName sql@domain.tld
	
	The script will make no changes to the LiteralAccountName and will seach for all 
	services using a StartName of "sql@domain.tld".
	
	This means, the script will not find services using a StartName of domain\sql or
	domain.tld\sql.
	
.EXAMPLE
	PS C:\PSScript > .\Get-ServiceWithAccountName.ps1 -AccountName svc_acct -ComputerName PVS
	
	The script will change "svc_acct" to "*svc_acct*" and will seach for all 
	services using a StartName that contains "svc_acct".

	If InputFIle is not used, the script will only search computers with "PVS" 
	in the DNSHostName.
	
	If InputFile is used, the script will only search computers with "PVS" in 
	the entries contained in the file.

.EXAMPLE
	PS C:\PSScript > .\Get-ServiceWithAccountName.ps1 -AccountName ctxadmin -Folder \\FileServer\ShareName
	
	The script will change "ctxadmin" to "*ctxadmin*".
	Output file will be saved in the path \\FileServer\ShareName
	
.EXAMPLE
	PS C:\PSScript > .\Get-ServiceWithAccountName.ps1 -AccountName svc_acct -ComputerName SQL -InputFile c:\Scripts\computers.txt
	
	The script will change "svc_acct" to "*svc_acct*" and will seach for all 
	services using a StartName that contains "svc_acct".

	The script will only search computers with "SQL" in the file computers.txt.
	
	InputFile causes the script to not check for or use the ActiveDirectory module.

.INPUTS
	None.  You cannot pipe objects to this script.
.OUTPUTS
	No objects are output from this script.  This script creates two texts files.
.NOTES
	NAME: Get-ServiceWithAccountName.ps1
	VERSION: 1.00
	AUTHOR: Carl Webster
	LASTEDIT: May 14, 2017
#>


#Created by Carl Webster, CTP 30-Mar-2017
#webster@carlwebster.com
#@carlwebster on Twitter
#http://www.CarlWebster.com

[CmdletBinding(SupportsShouldProcess = $False, ConfirmImpact = "None", DefaultParameterSetName = "AccountName") ]

Param(
	[parameter(ParameterSetName="AccountName",Mandatory=$True)] 
	[Alias("AN")]
	[ValidateNotNullOrEmpty()]
	[string]$AccountName,
	
	[parameter(ParameterSetName="LiteralAccountName",Mandatory=$True)] 
	[Alias("LAN")]
	[ValidateNotNullOrEmpty()]
	[string]$LiteralAccountName,

	[parameter(Mandatory=$False)] 
	[Alias("CN")]
	[string]$ComputerName,
	
	[parameter(Mandatory=$False)] 
	[Alias("IF")]
	[string]$InputFile="",
	
	[parameter(Mandatory=$False)] 
	[Alias("OU")]
	[string]$OrganizationalUnit="",
	
	[parameter(Mandatory=$False)] 
	[string]$Folder=""
	
	)

Set-StrictMode -Version 2

Write-Host "$(Get-Date): Setting up script"

If(![String]::IsNullOrEmpty($InputFile))
{
	Write-Host "$(Get-Date): Validating input file"
	If(!(Test-Path $InputFile))
	{
		Write-Error "Input file specified but $InputFile does not exist. Script cannot continue."
		Exit
	}
}

If($Folder -ne "")
{
	Write-Host "$(Get-Date): Testing folder path"
	#does it exist
	If(Test-Path $Folder -EA 0)
	{
		#it exists, now check to see if it is a folder and not a file
		If(Test-Path $Folder -pathType Container -EA 0)
		{
			#it exists and it is a folder
			Write-Host "$(Get-Date): Folder path $Folder exists and is a folder"
		}
		Else
		{
			#it exists but it is a file not a folder
			Write-Error "Folder $Folder is a file, not a folder. Script cannot continue"
			Exit
		}
	}
	Else
	{
		#does not exist
		Write-Error "Folder $Folder does not exist.  Script cannot continue"
		Exit
	}
}

#test to see if OrganizationalUnit is valid
If(![String]::IsNullOrEmpty($OrganizationalUnit))
{
	Write-Host "$(Get-Date): Validating Organnization Unit"
	try 
	{
		$results = Get-ADOrganizationalUnit -Identity $OrganizationalUnit
	} 
	
	catch
	{
		#does not exist
		Write-Error "Organization Unit $OrganizationalUnit does not exist.`n`nScript cannot continue`n`n"
		Exit
	}	
}

If($Folder -eq "")
{
	$pwdpath = $pwd.Path
}
Else
{
	$pwdpath = $Folder
}

If($pwdpath.EndsWith("\"))
{
	#remove the trailing \
	$pwdpath = $pwdpath.SubString(0, ($pwdpath.Length - 1))
}

Function Check-LoadedModule
#Function created by Jeff Wouters
#@JeffWouters on Twitter
#modified by Michael B. Smith to handle when the module doesn't exist on server
#modified by @andyjmorgan
#bug fixed by @schose
#bug fixed by Peter Bosen
#This Function handles all three scenarios:
#
# 1. Module is already imported into current session
# 2. Module is not already imported into current session, it does exists on the server and is imported
# 3. Module does not exist on the server

{
	Param([parameter(Mandatory = $True)][alias("Module")][string]$ModuleName)
	#following line changed at the recommendation of @andyjmorgan
	$LoadedModules = Get-Module |% { $_.Name.ToString() }
	#bug reported on 21-JAN-2013 by @schose 
	
	[string]$ModuleFound = ($LoadedModules -like "*$ModuleName*")
	If($ModuleFound -ne $ModuleName) 
	{
		$module = Import-Module -Name $ModuleName -PassThru -EA 0
		If($module -and $?)
		{
			# module imported properly
			Return $True
		}
		Else
		{
			# module import failed
			Return $False
		}
	}
	Else
	{
		#module already imported into current session
		Return $True
	}
}

Function ProcessComputer 
{
	Param([string]$TmpComputerName)
	
	If(Test-Connection -ComputerName $TmpComputerName -quiet -EA 0)
	{
		If(![String]::IsNullOrEmpty($AccountName))
		{
			$Results = Get-WmiObject -ComputerName $TmpComputerName Win32_Service -EA 0 | Where-Object {$_.StartName -like $testname} | Select SystemName, Name, DisplayName, StartName
		}
		ElseIf(![String]::IsNullOrEmpty($LiteralAccountName))
		{
			$Results = Get-WmiObject -ComputerName $TmpComputerName Win32_Service -EA 0 | Where-Object {$_.StartName -eq $testname} | Select SystemName, Name, DisplayName, StartName
		}
		
		If($? -and $Null -ne $Results)
		{
			Write-Host "`tFound a match"
			$Script:AllMatches += $Results
		}
	}
	Else
	{
		Write-Host "`tComputer $($TmpComputerName) is not online"
		Out-File -FilePath $Filename2 -Append -InputObject "Computer $($TmpComputerName) was not online $(Get-Date)"
	}
}

#only check for the ActiveDirectory module if an InputFile was not entered
If([String]::IsNullOrEmpty($InputFile) -and !(Check-LoadedModule "ActiveDirectory"))
{
	Write-Host "Unable to run script, no ActiveDirectory module"
	Exit
}

[string]$Script:FileName = "$($pwdpath)\ServersWithServiceAccount.txt"
[string]$Script:FileName2 = "$($pwdpath)\ServersWithServiceAccountErrors.txt"

If(![String]::IsNullOrEmpty($ComputerName) -and [String]::IsNullOrEmpty($InputFile))
{
	#computer name but no input file
	Write-Host "$(Get-Date): Retrieving list of computers from Active Directory"
	$testname = "*$($ComputerName)*"
	If(![String]::IsNullOrEmpty($OrganizationalUnit))
	{
		$Computers = Get-AdComputer -filter {DNSHostName -like $testname} -SearchBase $OrganizationalUnit -SearchScope Subtree -properties DNSHostName, Name -EA 0 | Sort Name
	}
	Else
	{
		$Computers = Get-AdComputer -filter {DNSHostName -like $testname} -properties DNSHostName, Name -EA 0 | Sort Name
	}
}
ElseIf([String]::IsNullOrEmpty($ComputerName) -and ![String]::IsNullOrEmpty($InputFile))
{
	#input file but no computer name
	Write-Host "$(Get-Date): Retrieving list of computers from Input File"
	$Computers = Get-Content $InputFile
}
ElseIf(![String]::IsNullOrEmpty($ComputerName) -and ![String]::IsNullOrEmpty($InputFile))
{
	#both computer name and input file
	Write-Host "$(Get-Date): Retrieving list of computers from Input File"
	$testname = "*$($ComputerName)*"
	$Computers = Get-Content $InputFile | ? {$_ -like $testname}
}
Else
{
	Write-Host "$(Get-Date): Retrieving list of computers from Active Directory"
	If(![String]::IsNullOrEmpty($OrganizationalUnit))
	{
		$Computers = Get-AdComputer -filter {OperatingSystem -like "*server*"} -SearchBase $OrganizationalUnit -SearchScope Subtree -properties DNSHostName, Name -EA 0 | Sort Name
	}
	Else
	{
		$Computers = Get-AdComputer -filter {OperatingSystem -like "*server*"} -properties DNSHostName, Name -EA 0 | Sort Name
	}
}

If($? -and $Null -ne $Computers)
{
	If($Computers -is [array])
	{
		Write-Host "Found $($Computers.Count) servers to process"
	}
	Else
	{
		Write-Host "Found 1 server to process"
	}
	
	$startTime = Get-Date
	If(![String]::IsNullOrEmpty($AccountName))
	{
		$testname = "*$($AccountName)*"
	}
	ElseIf(![String]::IsNullOrEmpty($LiteralAccountName))
	{
		$testname = "$($LiteralAccountName)"
	}

	$Script:AllMatches = @()

	If(![String]::IsNullOrEmpty($InputFile))
	{
		ForEach($Computer in $Computers)
		{
			$TmpComputerName = $Computer
			Write-Host "Testing computer $($TmpComputerName)"
			ProcessComputer $TmpComputerName
		}
	}
	Else
	{
		ForEach($Computer in $Computers)
		{
			$TmpComputerName = $Computer.DNSHostName
			Write-Host "Testing computer $($TmpComputerName)"
			ProcessComputer $TmpComputerName
		}
	}

	$Script:AllMatches | ft SystemName, Name, DisplayName, StartName
	
	$Script:AllMatches | Out-String -width 200 | Out-File -FilePath $Script:FileName

	If(Test-Path "$($Script:FileName)")
	{
		Write-Host "$(Get-Date): $($Script:FileName) is ready for use"
	}
	If(Test-Path "$($Script:FileName2)")
	{
		Write-Host "$(Get-Date): $($Script:FileName2) is ready for use"
	}
	
	Write-Host "$(Get-Date): Script started: $($StartTime)"
	Write-Host "$(Get-Date): Script ended: $(Get-Date)"
	$runtime = $(Get-Date) - $StartTime
	$Str = [string]::format("{0} days, {1} hours, {2} minutes, {3}.{4} seconds", `
		$runtime.Days, `
		$runtime.Hours, `
		$runtime.Minutes, `
		$runtime.Seconds,
		$runtime.Milliseconds)
	Write-Host "$(Get-Date): Elapsed time: $($Str)"
	$runtime = $Null
}
Else
{
	If(![String]::IsNullOrEmpty($ComputerName) -and [String]::IsNullOrEmpty($InputFile))
	{
		#computer name but no input file
		Write-Host "Unable to retrieve a list of computers from Active Directory"
	}
	ElseIf([String]::IsNullOrEmpty($ComputerName) -and ![String]::IsNullOrEmpty($InputFile))
	{
		#input file but no computer name
		Write-Host "Unable to retrieve a list of computers from the Input File $InputFile"
	}
	ElseIf(![String]::IsNullOrEmpty($ComputerName) -and ![String]::IsNullOrEmpty($InputFile))
	{
		#computer name and input file
		Write-Host "Unable to retrieve a list of matching computers from the Input File $InputFile"
	}
	Else
	{
		Write-Host "Unable to retrieve a list of computers from Active Directory"
	}
}
# SIG # Begin signature block
# MIIf8QYJKoZIhvcNAQcCoIIf4jCCH94CAQExCzAJBgUrDgMCGgUAMGkGCisGAQQB
# gjcCAQSgWzBZMDQGCisGAQQBgjcCAR4wJgIDAQAABBAfzDtgWUsITrck0sYpfvNR
# AgEAAgEAAgEAAgEAAgEAMCEwCQYFKw4DAhoFAAQUGWaEfYfQ5GgReu6GlDAncMML
# EZSgghtYMIIDtzCCAp+gAwIBAgIQDOfg5RfYRv6P5WD8G/AwOTANBgkqhkiG9w0B
# AQUFADBlMQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMRkwFwYD
# VQQLExB3d3cuZGlnaWNlcnQuY29tMSQwIgYDVQQDExtEaWdpQ2VydCBBc3N1cmVk
# IElEIFJvb3QgQ0EwHhcNMDYxMTEwMDAwMDAwWhcNMzExMTEwMDAwMDAwWjBlMQsw
# CQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQgSW5jMRkwFwYDVQQLExB3d3cu
# ZGlnaWNlcnQuY29tMSQwIgYDVQQDExtEaWdpQ2VydCBBc3N1cmVkIElEIFJvb3Qg
# Q0EwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQCtDhXO5EOAXLGH87dg
# +XESpa7cJpSIqvTO9SA5KFhgDPiA2qkVlTJhPLWxKISKityfCgyDF3qPkKyK53lT
# XDGEKvYPmDI2dsze3Tyoou9q+yHyUmHfnyDXH+Kx2f4YZNISW1/5WBg1vEfNoTb5
# a3/UsDg+wRvDjDPZ2C8Y/igPs6eD1sNuRMBhNZYW/lmci3Zt1/GiSw0r/wty2p5g
# 0I6QNcZ4VYcgoc/lbQrISXwxmDNsIumH0DJaoroTghHtORedmTpyoeb6pNnVFzF1
# roV9Iq4/AUaG9ih5yLHa5FcXxH4cDrC0kqZWs72yl+2qp/C3xag/lRbQ/6GW6whf
# GHdPAgMBAAGjYzBhMA4GA1UdDwEB/wQEAwIBhjAPBgNVHRMBAf8EBTADAQH/MB0G
# A1UdDgQWBBRF66Kv9JLLgjEtUYunpyGd823IDzAfBgNVHSMEGDAWgBRF66Kv9JLL
# gjEtUYunpyGd823IDzANBgkqhkiG9w0BAQUFAAOCAQEAog683+Lt8ONyc3pklL/3
# cmbYMuRCdWKuh+vy1dneVrOfzM4UKLkNl2BcEkxY5NM9g0lFWJc1aRqoR+pWxnmr
# EthngYTffwk8lOa4JiwgvT2zKIn3X/8i4peEH+ll74fg38FnSbNd67IJKusm7Xi+
# fT8r87cmNW1fiQG2SVufAQWbqz0lwcy2f8Lxb4bG+mRo64EtlOtCt/qMHt1i8b5Q
# Z7dsvfPxH2sMNgcWfzd8qVttevESRmCD1ycEvkvOl77DZypoEd+A5wwzZr8TDRRu
# 838fYxAe+o0bJW1sj6W3YQGx0qMmoRBxna3iw/nDmVG3KwcIzi7mULKn+gpFL6Lw
# 8jCCBSYwggQOoAMCAQICEAZY+tvHeDVvdG/HsafuSKwwDQYJKoZIhvcNAQELBQAw
# cjELMAkGA1UEBhMCVVMxFTATBgNVBAoTDERpZ2lDZXJ0IEluYzEZMBcGA1UECxMQ
# d3d3LmRpZ2ljZXJ0LmNvbTExMC8GA1UEAxMoRGlnaUNlcnQgU0hBMiBBc3N1cmVk
# IElEIENvZGUgU2lnbmluZyBDQTAeFw0xOTEwMTUwMDAwMDBaFw0yMDEyMDQxMjAw
# MDBaMGMxCzAJBgNVBAYTAlVTMRIwEAYDVQQIEwlUZW5uZXNzZWUxEjAQBgNVBAcT
# CVR1bGxhaG9tYTEVMBMGA1UEChMMQ2FybCBXZWJzdGVyMRUwEwYDVQQDEwxDYXJs
# IFdlYnN0ZXIwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDCib5DeGTG
# 3J70a2CA8i9n+dPsDklvWpkUTAuZesMTdgYYYKJTsaaNY/UEAHlJukWzaoFQUJc8
# cf5mUa48zGHKjIsFRJtv1YjaeoJzdLBWiqSaI6m3Ttkj8YqvAVj7U3wDNc30gWgU
# eJwPQs2+Ge6tVHRx7/Knzu12RkJ/fEUwoqwHyL5ezfBHfIf3AiukAxRMKrsqGMPI
# 20y/mc8oiwTuyCG9vieR9+V+iq+ATGgxxb+TOzRoxyFsYOcqnGv3iHqNr74y+rfC
# /HfkieCRmkwh0ss4EVnKIJMefWIlkH3HPirYn+4wmeTKQZmtIq0oEbJlXsSryOXW
# i/NjGfe2xXENAgMBAAGjggHFMIIBwTAfBgNVHSMEGDAWgBRaxLl7KgqjpepxA8Bg
# +S32ZXUOWDAdBgNVHQ4EFgQUqRd4UyWyhbxwBUPJhcJf/q5IdaQwDgYDVR0PAQH/
# BAQDAgeAMBMGA1UdJQQMMAoGCCsGAQUFBwMDMHcGA1UdHwRwMG4wNaAzoDGGL2h0
# dHA6Ly9jcmwzLmRpZ2ljZXJ0LmNvbS9zaGEyLWFzc3VyZWQtY3MtZzEuY3JsMDWg
# M6Axhi9odHRwOi8vY3JsNC5kaWdpY2VydC5jb20vc2hhMi1hc3N1cmVkLWNzLWcx
# LmNybDBMBgNVHSAERTBDMDcGCWCGSAGG/WwDATAqMCgGCCsGAQUFBwIBFhxodHRw
# czovL3d3dy5kaWdpY2VydC5jb20vQ1BTMAgGBmeBDAEEATCBhAYIKwYBBQUHAQEE
# eDB2MCQGCCsGAQUFBzABhhhodHRwOi8vb2NzcC5kaWdpY2VydC5jb20wTgYIKwYB
# BQUHMAKGQmh0dHA6Ly9jYWNlcnRzLmRpZ2ljZXJ0LmNvbS9EaWdpQ2VydFNIQTJB
# c3N1cmVkSURDb2RlU2lnbmluZ0NBLmNydDAMBgNVHRMBAf8EAjAAMA0GCSqGSIb3
# DQEBCwUAA4IBAQBMkLEdY3RRV97ghwUHUZlBdZ9dFFjBx6WB3rAGTeS2UaGlZuwj
# 2zigbOf8TAJGXiT4pBIZ17X01rpbopIeGGW6pNEUIQQlqaXHQUsY8kbjwVVSdQki
# c1ZwNJoGdgsE50yxPYq687+LR1rgViKuhkTN79ffM5kuqofxoGByxgbinRbC3PQp
# H3U6c1UhBRYAku/l7ev0dFvibUlRgV4B6RjQBylZ09+rcXeT+GKib13Ma6bjcKTq
# qsf9PgQ6P5/JNnWdy19r10SFlsReHElnnSJeRLAptk9P7CRU5/cMkI7CYAR0GWdn
# e1/Kdz6FwvSJl0DYr1p0utdyLRVpgHKG30bTMIIFMDCCBBigAwIBAgIQBAkYG1/V
# u2Z1U0O1b5VQCDANBgkqhkiG9w0BAQsFADBlMQswCQYDVQQGEwJVUzEVMBMGA1UE
# ChMMRGlnaUNlcnQgSW5jMRkwFwYDVQQLExB3d3cuZGlnaWNlcnQuY29tMSQwIgYD
# VQQDExtEaWdpQ2VydCBBc3N1cmVkIElEIFJvb3QgQ0EwHhcNMTMxMDIyMTIwMDAw
# WhcNMjgxMDIyMTIwMDAwWjByMQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNl
# cnQgSW5jMRkwFwYDVQQLExB3d3cuZGlnaWNlcnQuY29tMTEwLwYDVQQDEyhEaWdp
# Q2VydCBTSEEyIEFzc3VyZWQgSUQgQ29kZSBTaWduaW5nIENBMIIBIjANBgkqhkiG
# 9w0BAQEFAAOCAQ8AMIIBCgKCAQEA+NOzHH8OEa9ndwfTCzFJGc/Q+0WZsTrbRPV/
# 5aid2zLXcep2nQUut4/6kkPApfmJ1DcZ17aq8JyGpdglrA55KDp+6dFn08b7KSfH
# 03sjlOSRI5aQd4L5oYQjZhJUM1B0sSgmuyRpwsJS8hRniolF1C2ho+mILCCVrhxK
# hwjfDPXiTWAYvqrEsq5wMWYzcT6scKKrzn/pfMuSoeU7MRzP6vIK5Fe7SrXpdOYr
# /mzLfnQ5Ng2Q7+S1TqSp6moKq4TzrGdOtcT3jNEgJSPrCGQ+UpbB8g8S9MWOD8Gi
# 6CxR93O8vYWxYoNzQYIH5DiLanMg0A9kczyen6Yzqf0Z3yWT0QIDAQABo4IBzTCC
# AckwEgYDVR0TAQH/BAgwBgEB/wIBADAOBgNVHQ8BAf8EBAMCAYYwEwYDVR0lBAww
# CgYIKwYBBQUHAwMweQYIKwYBBQUHAQEEbTBrMCQGCCsGAQUFBzABhhhodHRwOi8v
# b2NzcC5kaWdpY2VydC5jb20wQwYIKwYBBQUHMAKGN2h0dHA6Ly9jYWNlcnRzLmRp
# Z2ljZXJ0LmNvbS9EaWdpQ2VydEFzc3VyZWRJRFJvb3RDQS5jcnQwgYEGA1UdHwR6
# MHgwOqA4oDaGNGh0dHA6Ly9jcmw0LmRpZ2ljZXJ0LmNvbS9EaWdpQ2VydEFzc3Vy
# ZWRJRFJvb3RDQS5jcmwwOqA4oDaGNGh0dHA6Ly9jcmwzLmRpZ2ljZXJ0LmNvbS9E
# aWdpQ2VydEFzc3VyZWRJRFJvb3RDQS5jcmwwTwYDVR0gBEgwRjA4BgpghkgBhv1s
# AAIEMCowKAYIKwYBBQUHAgEWHGh0dHBzOi8vd3d3LmRpZ2ljZXJ0LmNvbS9DUFMw
# CgYIYIZIAYb9bAMwHQYDVR0OBBYEFFrEuXsqCqOl6nEDwGD5LfZldQ5YMB8GA1Ud
# IwQYMBaAFEXroq/0ksuCMS1Ri6enIZ3zbcgPMA0GCSqGSIb3DQEBCwUAA4IBAQA+
# 7A1aJLPzItEVyCx8JSl2qB1dHC06GsTvMGHXfgtg/cM9D8Svi/3vKt8gVTew4fbR
# knUPUbRupY5a4l4kgU4QpO4/cY5jDhNLrddfRHnzNhQGivecRk5c/5CxGwcOkRX7
# uq+1UcKNJK4kxscnKqEpKBo6cSgCPC6Ro8AlEeKcFEehemhor5unXCBc2XGxDI+7
# qPjFEmifz0DLQESlE/DmZAwlCEIysjaKJAL+L3J+HNdJRZboWR3p+nRka7LrZkPa
# s7CM1ekN3fYBIM6ZMWM9CBoYs4GbT8aTEAb8B4H6i9r5gkn3Ym6hU/oSlBiFLpKR
# 6mhsRDKyZqHnGKSaZFHvMIIGajCCBVKgAwIBAgIQAwGaAjr/WLFr1tXq5hfwZjAN
# BgkqhkiG9w0BAQUFADBiMQswCQYDVQQGEwJVUzEVMBMGA1UEChMMRGlnaUNlcnQg
# SW5jMRkwFwYDVQQLExB3d3cuZGlnaWNlcnQuY29tMSEwHwYDVQQDExhEaWdpQ2Vy
# dCBBc3N1cmVkIElEIENBLTEwHhcNMTQxMDIyMDAwMDAwWhcNMjQxMDIyMDAwMDAw
# WjBHMQswCQYDVQQGEwJVUzERMA8GA1UEChMIRGlnaUNlcnQxJTAjBgNVBAMTHERp
# Z2lDZXJ0IFRpbWVzdGFtcCBSZXNwb25kZXIwggEiMA0GCSqGSIb3DQEBAQUAA4IB
# DwAwggEKAoIBAQCjZF38fLPggjXg4PbGKuZJdTvMbuBTqZ8fZFnmfGt/a4ydVfiS
# 457VWmNbAklQ2YPOb2bu3cuF6V+l+dSHdIhEOxnJ5fWRn8YUOawk6qhLLJGJzF4o
# 9GS2ULf1ErNzlgpno75hn67z/RJ4dQ6mWxT9RSOOhkRVfRiGBYxVh3lIRvfKDo2n
# 3k5f4qi2LVkCYYhhchhoubh87ubnNC8xd4EwH7s2AY3vJ+P3mvBMMWSN4+v6GYeo
# fs/sjAw2W3rBerh4x8kGLkYQyI3oBGDbvHN0+k7Y/qpA8bLOcEaD6dpAoVk62RUJ
# V5lWMJPzyWHM0AjMa+xiQpGsAsDvpPCJEY93AgMBAAGjggM1MIIDMTAOBgNVHQ8B
# Af8EBAMCB4AwDAYDVR0TAQH/BAIwADAWBgNVHSUBAf8EDDAKBggrBgEFBQcDCDCC
# Ab8GA1UdIASCAbYwggGyMIIBoQYJYIZIAYb9bAcBMIIBkjAoBggrBgEFBQcCARYc
# aHR0cHM6Ly93d3cuZGlnaWNlcnQuY29tL0NQUzCCAWQGCCsGAQUFBwICMIIBVh6C
# AVIAQQBuAHkAIAB1AHMAZQAgAG8AZgAgAHQAaABpAHMAIABDAGUAcgB0AGkAZgBp
# AGMAYQB0AGUAIABjAG8AbgBzAHQAaQB0AHUAdABlAHMAIABhAGMAYwBlAHAAdABh
# AG4AYwBlACAAbwBmACAAdABoAGUAIABEAGkAZwBpAEMAZQByAHQAIABDAFAALwBD
# AFAAUwAgAGEAbgBkACAAdABoAGUAIABSAGUAbAB5AGkAbgBnACAAUABhAHIAdAB5
# ACAAQQBnAHIAZQBlAG0AZQBuAHQAIAB3AGgAaQBjAGgAIABsAGkAbQBpAHQAIABs
# AGkAYQBiAGkAbABpAHQAeQAgAGEAbgBkACAAYQByAGUAIABpAG4AYwBvAHIAcABv
# AHIAYQB0AGUAZAAgAGgAZQByAGUAaQBuACAAYgB5ACAAcgBlAGYAZQByAGUAbgBj
# AGUALjALBglghkgBhv1sAxUwHwYDVR0jBBgwFoAUFQASKxOYspkH7R7for5XDStn
# As0wHQYDVR0OBBYEFGFaTSS2STKdSip5GoNL9B6Jwcp9MH0GA1UdHwR2MHQwOKA2
# oDSGMmh0dHA6Ly9jcmwzLmRpZ2ljZXJ0LmNvbS9EaWdpQ2VydEFzc3VyZWRJRENB
# LTEuY3JsMDigNqA0hjJodHRwOi8vY3JsNC5kaWdpY2VydC5jb20vRGlnaUNlcnRB
# c3N1cmVkSURDQS0xLmNybDB3BggrBgEFBQcBAQRrMGkwJAYIKwYBBQUHMAGGGGh0
# dHA6Ly9vY3NwLmRpZ2ljZXJ0LmNvbTBBBggrBgEFBQcwAoY1aHR0cDovL2NhY2Vy
# dHMuZGlnaWNlcnQuY29tL0RpZ2lDZXJ0QXNzdXJlZElEQ0EtMS5jcnQwDQYJKoZI
# hvcNAQEFBQADggEBAJ0lfhszTbImgVybhs4jIA+Ah+WI//+x1GosMe06FxlxF82p
# G7xaFjkAneNshORaQPveBgGMN/qbsZ0kfv4gpFetW7easGAm6mlXIV00Lx9xsIOU
# GQVrNZAQoHuXx/Y/5+IRQaa9YtnwJz04HShvOlIJ8OxwYtNiS7Dgc6aSwNOOMdgv
# 420XEwbu5AO2FKvzj0OncZ0h3RTKFV2SQdr5D4HRmXQNJsQOfxu19aDxxncGKBXp
# 2JPlVRbwuwqrHNtcSCdmyKOLChzlldquxC5ZoGHd2vNtomHpigtt7BIYvfdVVEAD
# kitrwlHCCkivsNRu4PQUCjob4489yq9qjXvc2EQwggbNMIIFtaADAgECAhAG/fkD
# lgOt6gAK6z8nu7obMA0GCSqGSIb3DQEBBQUAMGUxCzAJBgNVBAYTAlVTMRUwEwYD
# VQQKEwxEaWdpQ2VydCBJbmMxGTAXBgNVBAsTEHd3dy5kaWdpY2VydC5jb20xJDAi
# BgNVBAMTG0RpZ2lDZXJ0IEFzc3VyZWQgSUQgUm9vdCBDQTAeFw0wNjExMTAwMDAw
# MDBaFw0yMTExMTAwMDAwMDBaMGIxCzAJBgNVBAYTAlVTMRUwEwYDVQQKEwxEaWdp
# Q2VydCBJbmMxGTAXBgNVBAsTEHd3dy5kaWdpY2VydC5jb20xITAfBgNVBAMTGERp
# Z2lDZXJ0IEFzc3VyZWQgSUQgQ0EtMTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCC
# AQoCggEBAOiCLZn5ysJClaWAc0Bw0p5WVFypxNJBBo/JM/xNRZFcgZ/tLJz4Flnf
# nrUkFcKYubR3SdyJxArar8tea+2tsHEx6886QAxGTZPsi3o2CAOrDDT+GEmC/sfH
# MUiAfB6iD5IOUMnGh+s2P9gww/+m9/uizW9zI/6sVgWQ8DIhFonGcIj5BZd9o8dD
# 3QLoOz3tsUGj7T++25VIxO4es/K8DCuZ0MZdEkKB4YNugnM/JksUkK5ZZgrEjb7S
# zgaurYRvSISbT0C58Uzyr5j79s5AXVz2qPEvr+yJIvJrGGWxwXOt1/HYzx4KdFxC
# uGh+t9V3CidWfA9ipD8yFGCV/QcEogkCAwEAAaOCA3owggN2MA4GA1UdDwEB/wQE
# AwIBhjA7BgNVHSUENDAyBggrBgEFBQcDAQYIKwYBBQUHAwIGCCsGAQUFBwMDBggr
# BgEFBQcDBAYIKwYBBQUHAwgwggHSBgNVHSAEggHJMIIBxTCCAbQGCmCGSAGG/WwA
# AQQwggGkMDoGCCsGAQUFBwIBFi5odHRwOi8vd3d3LmRpZ2ljZXJ0LmNvbS9zc2wt
# Y3BzLXJlcG9zaXRvcnkuaHRtMIIBZAYIKwYBBQUHAgIwggFWHoIBUgBBAG4AeQAg
# AHUAcwBlACAAbwBmACAAdABoAGkAcwAgAEMAZQByAHQAaQBmAGkAYwBhAHQAZQAg
# AGMAbwBuAHMAdABpAHQAdQB0AGUAcwAgAGEAYwBjAGUAcAB0AGEAbgBjAGUAIABv
# AGYAIAB0AGgAZQAgAEQAaQBnAGkAQwBlAHIAdAAgAEMAUAAvAEMAUABTACAAYQBu
# AGQAIAB0AGgAZQAgAFIAZQBsAHkAaQBuAGcAIABQAGEAcgB0AHkAIABBAGcAcgBl
# AGUAbQBlAG4AdAAgAHcAaABpAGMAaAAgAGwAaQBtAGkAdAAgAGwAaQBhAGIAaQBs
# AGkAdAB5ACAAYQBuAGQAIABhAHIAZQAgAGkAbgBjAG8AcgBwAG8AcgBhAHQAZQBk
# ACAAaABlAHIAZQBpAG4AIABiAHkAIAByAGUAZgBlAHIAZQBuAGMAZQAuMAsGCWCG
# SAGG/WwDFTASBgNVHRMBAf8ECDAGAQH/AgEAMHkGCCsGAQUFBwEBBG0wazAkBggr
# BgEFBQcwAYYYaHR0cDovL29jc3AuZGlnaWNlcnQuY29tMEMGCCsGAQUFBzAChjdo
# dHRwOi8vY2FjZXJ0cy5kaWdpY2VydC5jb20vRGlnaUNlcnRBc3N1cmVkSURSb290
# Q0EuY3J0MIGBBgNVHR8EejB4MDqgOKA2hjRodHRwOi8vY3JsMy5kaWdpY2VydC5j
# b20vRGlnaUNlcnRBc3N1cmVkSURSb290Q0EuY3JsMDqgOKA2hjRodHRwOi8vY3Js
# NC5kaWdpY2VydC5jb20vRGlnaUNlcnRBc3N1cmVkSURSb290Q0EuY3JsMB0GA1Ud
# DgQWBBQVABIrE5iymQftHt+ivlcNK2cCzTAfBgNVHSMEGDAWgBRF66Kv9JLLgjEt
# UYunpyGd823IDzANBgkqhkiG9w0BAQUFAAOCAQEARlA+ybcoJKc4HbZbKa9Sz1Lp
# MUerVlx71Q0LQbPv7HUfdDjyslxhopyVw1Dkgrkj0bo6hnKtOHisdV0XFzRyR4WU
# VtHruzaEd8wkpfMEGVWp5+Pnq2LN+4stkMLA0rWUvV5PsQXSDj0aqRRbpoYxYqio
# M+SbOafE9c4deHaUJXPkKqvPnHZL7V/CSxbkS3BMAIke/MV5vEwSV/5f4R68Al2o
# /vsHOE8Nxl2RuQ9nRc3Wg+3nkg2NsWmMT/tZ4CMP0qquAHzunEIOz5HXJ7cW7g/D
# vXwKoO4sCFWFIrjrGBpN/CohrUkxg0eVd3HcsRtLSxwQnHcUwZ1PL1qVCCkQJjGC
# BAMwggP/AgEBMIGGMHIxCzAJBgNVBAYTAlVTMRUwEwYDVQQKEwxEaWdpQ2VydCBJ
# bmMxGTAXBgNVBAsTEHd3dy5kaWdpY2VydC5jb20xMTAvBgNVBAMTKERpZ2lDZXJ0
# IFNIQTIgQXNzdXJlZCBJRCBDb2RlIFNpZ25pbmcgQ0ECEAZY+tvHeDVvdG/Hsafu
# SKwwCQYFKw4DAhoFAKBAMBkGCSqGSIb3DQEJAzEMBgorBgEEAYI3AgEEMCMGCSqG
# SIb3DQEJBDEWBBSlf3eAjCQ8DD4IxgMVqUfvogiKKDANBgkqhkiG9w0BAQEFAASC
# AQAJTlNgkD7QJIdDPFBdQ/rr3AoM9oThQbQv9SkUn9xC0zQROOWKuXAooa16CuqE
# K0PIaPF6YFO3bG11xhmT8p3bNS8NhJ93udTlrR8SiXZXbvDdG8DYwEqCaUyZV08q
# 42T0obu7j1V4fF0WgUNxwnar4idOywDRn5v0Pflmh60UdC8ZGrnIn+up/soQhlzX
# i9ZHry+vM0ysSK+GJsJfHLvaBRJspJoORNdrkkUN946MDd0tf5Nz8hMaZF9Yvw+N
# lS8QvURhp1gNa2hoZnQEydS+hk5lpX4gt9BvlRL5OslSSLtVA9vv/eqNjRAMksPN
# +OUl+UiRU82ZoJX0peatRrVyoYICDzCCAgsGCSqGSIb3DQEJBjGCAfwwggH4AgEB
# MHYwYjELMAkGA1UEBhMCVVMxFTATBgNVBAoTDERpZ2lDZXJ0IEluYzEZMBcGA1UE
# CxMQd3d3LmRpZ2ljZXJ0LmNvbTEhMB8GA1UEAxMYRGlnaUNlcnQgQXNzdXJlZCBJ
# RCBDQS0xAhADAZoCOv9YsWvW1ermF/BmMAkGBSsOAwIaBQCgXTAYBgkqhkiG9w0B
# CQMxCwYJKoZIhvcNAQcBMBwGCSqGSIb3DQEJBTEPFw0yMDEwMzExMzI1NDRaMCMG
# CSqGSIb3DQEJBDEWBBStL70L9jlXW9qJyVI+tDi8/1f+0DANBgkqhkiG9w0BAQEF
# AASCAQAns+WTZTFO1Br0RBd6iFo/++9jGQDHhYQYtOD54MFvi+bc6EEg+xgf23cd
# 1fjmEl6XylI9TFJxxr8VsXEiN2wFlmKdVEnnNqw2NJ4N+oHh9XxPxG9MarxNR884
# DvqdPP3HGBtq3o/Rqyz+zWnzaYQr2SD++E+SMCxKowyym/0txaAgMXt1PqWonka/
# yiyKHmNTbNmmibMWfPoth2w7wzpYja0Z48EJ/GSz7W/QuE1Ab9WQ2HER70L1PNwd
# pfcnpIXFhaT35kdc4snKw6rywcncgF3r783CiprXHhpZJurrV7x58TF3Hn9mK/AW
# xwjvGDRI+WCe3DrthZbICIKb127d
# SIG # End signature block
