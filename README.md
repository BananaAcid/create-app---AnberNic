# Create an App for Anbernic devices
like for the RG40XXV.

1. From the SSD APPS/clock folder,
  copy the sdl2.zip and the font/ folder.
  (You could also copy all apps to a local folder, for your AI TOOL to be able to read all these files as examples for a better outcome.
  
  - You could also tell your AI TOOL, im using OpenCode here, to get these files by connecting to the device ip using SSH:
    ```
    Connect using ssh://root:root@<DEVICE IP>/mnt/mmc/Roms/APPS/Clock and save sdl2.zip and the fonts folder locally.
    ```
  - Using windows, use Powershell. Something like this will upload the files:
    <details>
    <summary>Code</summary>
      
    ```powershell
    $DeviceIP = "10.0.0.92"  # this must be the correct IP to the device
    $secpasswd = ConvertTo-SecureString "root" -AsPlainText -Force
    $Credentials = New-Object System.Management.Automation.PSCredential("root", $secpasswd)
    
    $SFTPSession = New-SFTPSession -ComputerName $DeviceIP -Credential $Credentials -AcceptKey
    
    $files = @(
        @{Local="D:\temp\APPS - AnberNic - RG40XXV\story\app.py"; Remote="/mnt/mmc/Roms/APPS/story/"},
        @{Local="D:\temp\APPS - AnberNic - RG40XXV\story\graphic.py"; Remote="/mnt/mmc/Roms/APPS/story/"},
        @{Local="D:\temp\APPS - AnberNic - RG40XXV\story\lang\de_DE.json"; Remote="/mnt/mmc/Roms/APPS/story/lang/"},
        @{Local="D:\temp\APPS - AnberNic - RG40XXV\story\lang\en_US.json"; Remote="/mnt/mmc/Roms/APPS/story/lang/"}
    )
    
    foreach ($file in $files) {
        Write-Host "Uploading $($file.Local)..."
        Set-SFTPItem -SessionId $SFTPSession.SessionId -Path $file.Local -Destination $file.Remote -Force
    }
    
    Remove-SFTPSession -SessionId $SFTPSession.SessionId
    Write-Host "Done!"
    ```
    </details>

2. Then in PLAN-mode in your AI TOOL, tell it to read the `CREATE_APP_AGENT.md` and what your app should be about.
  ```
  Read the `CREATE_APP_AGENT.md`.
  Create an app, that does the following:
  ...
  ```

3. To test the app, let the AI TOOL run it locally, or upload to the APPS folder.

4. Finally, let the AI TOOL upload your app to the APPS folder
  ... to run: Device > App Center > Apps > TF-1 > select your app in the list (the <app_name>.sh is what you select in the list, with its icon shown to the right)
