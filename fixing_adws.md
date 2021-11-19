# Fixing error `Unable to find a default server with Active Directory Web Services running` on Windows Server Core 2019

I retrieved the above mentioned error message of `Unable to find a default server with Active Directory Web Services running`, when adding new AD users on a Domain Controller (Windows Server Core 2019) via powershell.

The issued command was:

```powershell
$pw = "password" | ConvertTo-SecureString -AsPlainText -Force
New-ADUser -Name username -GivenName forename -Surname surname -AccountPassword $pw -ChangePasswordAtLogon $False -Enabled $True
```

However, `Get-Service ADWS | Format-List -Property *` revealed, that the ADWS service is running correctly.
Hence, the problem must be something else.

Then, I found the `-Server` option of the `New-ADUser` command.
Adding it as `-Server localhost` fixed the problem.

So, the whole working command is thus:

```powershell
$pw = "password" | ConvertTo-SecureString -AsPlainText -Force
New-ADUser -Name username -GivenName forename -Surname surname -AccountPassword $pw -ChangePasswordAtLogon $False -Enabled $True -Server localhost
```