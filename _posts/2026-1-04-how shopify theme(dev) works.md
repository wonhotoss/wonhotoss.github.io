---
layout: post
title: "How Shopify theme(and it`s dev env) works"
tags: 
    - Shopify
    - Liquid
    - Javascript
    - CSS
---

Illustrating How Shopify theme and it`s dev environment works.

For the sake of simplicity, We will make world-class simple theme, "Hello Shopify theme".

env
    - dev store
    - vs code
    - shopify cli

create smallest theme    

CLI, push, pull, what is static and what is dynamic, and how it is rendered

what is template(idnex.json)

add section
    what is a section?
    create section file
    add dynamic section to template via online editor
    check from online editor
    add static section to template via index.json
    check from online editor
    check addible and removable of static / dynamic sections

    add section settings and apply when rendering

add blocks
    what is a block?
    add block and render with CSS in assets

add snippets
    what is a snippet?
    use snippet on duplicates
    if error cones out, try restart dev env. `Liquid error (sections/section-dynamic line 12): Could not find asset snippets/info-panel.liquid`



    
    

    



---------------

{% include mermaidInPost.html %}
<script src="../assets/scripts/mermaid.min.js"></script>

My Troubleshooting record from company project. 

- Product stack
    - Unity3D + Windows standalone build
    - [NSIS Portable](https://portableapps.com/apps/development/nsis_portable)
    - [Windows Deep Linking](https://assetstore.unity.com/packages/tools/integration/deep-linking-for-windows-standalone-exe-264033)

- Why we use `deep linking`
    - Our app needs users to log in with IDPW or SSO, includes Google login. AFAIK, Google login must be performed on a Google domain, like google.com or their native app. Their SDK is just a way to reach the site. And the only site we can reach to in Widows is google.com. So we build our own login web page, conduct every SSO process on it, and send back access token to our app for further requests.

<div class="mermaid">
    sequenceDiagram
    app->>+web browser: open login page
    web browser->>+server: login(IDPW or SSO token)
    server->>+web browser: access token
    web browser->>+app: deep link with access token
    app->>+server: authorized access with access token
</div>

<br>
<br>


- Why Unity doens`t support deep linking on Windows(Why we use Windows Deep Linking package)
    - `Deep Linking` is a way to send data app-to-app, so it must be hosted by OS itself like a service, since it should wake up target app if inactive.
    - A Unity standalone Windows build is basically portable — it isn’t an installed application, so it can’t register itself with the OS. It`s different from Android or IOS build, including manifest and Installation process. OS never know where the app is and how to call it. 
    - So we use `Windows Deep Linking` package from the Asset store.

<br>
<br>

- How the package(Windows deep linking, `WDL` below) works
    1. Deep linking needs `CustomURI` entry in the registry. Normally this entry stores the keyword and app executable path. and the OS call the app when deep link requested with the keyword.This is the standard way deep linking works. 
    2. When our app uses `WDL`, it stores CustomURI on the fly, but it stores temporary script and the path of the script is stored in CustomURI entry instead on app path. the script is excuted once the CustomURI invoked.
    3. The script contains our app path. And it accepts Deep link queries(parametets?) from OS when excuted. The script stores queries somewhere, and call our app.
    4. Our app gets focus by the script. It reads parameters from promised place. Deep linking is accomplished.
    5. Since the polling way above, WDL can simulate deep llinking even in Unity editor.

<br>
<br>

- Problem

    ![running](/assets/images/another-instance-is-already-running.png)

    - when my app completes the login and tries to send the access token back, the OS opens a new app instance, instead of focus the one already exists. Since we enabled `force single instance`, it drops error dialog and quit. Thanks to `WDL`, original app instance reads stored query params when it gets focus manually. But we want to everything looks fine without any error.

    ![single](/assets/images/force-single-instance.png)    

    - We use NSIS to make instaer package. This case happens only on the execution from `Run my app` button in the last dialog of installer.

    ![installation](/assets/images/run-after-installation.png)

<br>
<br>

- reason
    - permission issue. [ChatGPT](https://chatgpt.com/share/68ea9304-9860-800b-886e-5a1dba6d7d32)
    - Our installer requires admin permission since it installs firewall exception. Conseqently, the app process launched by installer inherits admin permission from installer.
    - Our app opens login page using UnityEngine.ApplicationOpenURL and starts WDL process. It installs temporary script and waits for custom URI invoked as mentioned above. But both opening Web browser and invoking custom URI are conducted by the OS, we lose admin permission during the process. The temporary script is excuted with user permission.
    - The temporary script queries ongoing process with given name, but it fails to find app process under admin permission. It spawns new process instead activating existing one.

<br>
<br>

- solution
    - Launch app with user permission. Thanks to [ShallExecAsUser](https://nsis.sourceforge.io/ShellExecAsUser_plug-in)

<br>
<br>

- rejected alternatives
    - User-level installer
        - Anyway we need admin permission because we need firewall exceptions.
    - Multiple instance + Manual process query and focus
        - If another process instance exist, yield focus and quit. But it will suffer the same problem as the script.    
    - Use customURI as usual way(calling app), instead of calling script querying process
        - We need two customURI - one for script to store info, another one for wake up process, since Unity doesnt pass pass query parameters. Too complex.
    - Fixing temporary script
        - Use process ID, not appPath, to avoid process query.
            - It sounds like it might work(not tested). But we will need multiple instances and manual process someday, So it would be good to fix permission issue here.
        