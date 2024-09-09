# MITRE Technique T1546.002

{% embed url="https://attack.mitre.org/techniques/T1546/002/" %}

{% @github-files/github-code-block %}

```splunk-spl
index=sysmon (EventCode=13 Details="c:\\windows\\system32\\*.scr" TargetObject="*\\Control Panel\\Desktop\\SCRNSAVE.EXE" ) OR (EventCode=1 CommandLine="reg*\ add *\\Control Panel\\Desktop\" /v SCRNSAVE.EXE /t REG_SZ /d *C:\\WINDOWS\\System32\\*.scr*") 
| rex field=CommandLine "/d\ \"(?<TargetFilename>\S+)\"" 
| eval TargetFilename=coalesce(TargetFilename, Details)
| eval TargetFilename=lower(TargetFilename)
| mvexpand User
| search NOT User="NOT_TRANSLATED"
| join TargetFilename [ search index=sysmon EventCode=11 TargetFilename="C:\\Windows\\System32*.scr" | eval TargetFilename=lower(TargetFilename) | fields TargetFilename Image ]
| table UtcTime EventCode Sid ProcessGuid ProcessId User ComputerName TargetFilename Image
```
