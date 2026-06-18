# Create an App for Anbernic devices
like for the RG40XXV.

1. From the SSD APPS/clock folder,
  copy the sdl2.zip and the font/ folder.
  (You could also copy all apps to a local folder, for your AI TOOL to be able to read all these files as examples for a better outcome.
  
  - You could also tell your AI TOOL, im using OpenCode here, to get these files by connecting to the device ip using SSH:
    ```
    Connect using ssh://root:root@<DEVICE IP>/mnt/mmc/Roms/APPS/Clock and save sdl2.zip and the fonts folder locally.
    (If we are using windows, use powershell and see https://stackoverflow.com/a/54925358/1644202 how to connect with a password)
    ```

2. Then in PLAN-mode in your AI TOOL, tell it to read the `CREATE_APP_AGENT.md` and what your app should be about.
  ```
  Read the `CREATE_APP_AGENT.md`.
  Create an app, that does the following:
  ...
  ```

3. To test the app, let the AI TOOL run it locally, or upload to the APPS folder.

4. Finally, let the AI TOOL upload your app to the APPS folder
  ... to run: Device > App Center > Apps > TF-1 > select your app in the list (the <app_name>.sh is what you select in the list, with its icon shown to the right)
