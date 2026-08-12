# HomebrewUpdate

HomebrewUpdate is a PS Vita user plugin that adds a complete update system to compatible homebrew applications.

It allows users to check for a new version directly from the application's LiveArea, read the update information, download the VPK through the Vita download manager, verify the downloaded file, and install the update.

The entire process is handled on the PS Vita. A computer is not required once HomebrewUpdate and a compatible application are installed.

> HomebrewUpdate only works with applications whose developers have added HomebrewUpdate support.

## Features

- Checks for updates directly from the application's LiveArea.
- Displays the new version and its change information before downloading.
- Supports VPK downloads over HTTP and HTTPS.
- Uses the PS Vita download manager.
- Supports Pause, Resume, and Cancel.
- Restores interrupted downloads after a reboot when possible.
- Verifies the downloaded file before installation.
- Checks the VPK size and SHA-1.
- Checks the internal VPK structure.
- Checks that the VPK belongs to the correct application.
- Checks that the version inside the VPK matches the version announced by the update server.
- Deletes invalid or incomplete update files automatically.
- Displays clear notifications during every stage of the update.
- Provides a global setting to enable or disable all homebrew updates.

## Requirements

- A hacked PS Vita or PS TV with taiHEN.
- A compatible application containing `sce_sys/homebrew_update.ini`.
- An Internet connection.

## Installation

1. Copy `HomebrewUpdate.suprx` to:

   ```text
   ur0:tai/HomebrewUpdate.suprx
   ```

2. Add the plugin under `*main` in `ur0:tai/config.txt`:

   ```ini
   *main
   ur0:tai/HomebrewUpdate.suprx
   ```

3. Reboot the console.

If a `*main` section already exists, add only the plugin path below the existing `*main` line. Do not create a second `*main` section.

## How to use it

1. Open the LiveArea of a compatible application.
2. Select the update button.
3. HomebrewUpdate checks the update server.
4. If a newer version is available, select **Download**.
5. Wait for the download and verification to finish.
6. When **Waiting to Install** appears, open the notification and confirm the installation.

## Enable or disable HomebrewUpdate

HomebrewUpdate creates the following global configuration file automatically:

```text
ux0:data/HomebrewUpdater/setting.ini
```

Its default content is:

```ini
enabled=true
```

You can open and edit this file with VitaShell, VitaTweakBox, or another file manager:

- `enabled=true` allows HomebrewUpdate to work normally.
- `enabled=false` blocks HomebrewUpdate globally for every compatible application.

The setting is checked twice:

1. **When the PS Vita starts:** if the value is already `false`, HomebrewUpdate stops before loading its hooks. If you later change it back to `true`, you must reboot the PS Vita to load the plugin hooks again.
2. **Each time an update is requested:** if HomebrewUpdate started with `true`, changing the value to `false` blocks new update attempts immediately without a reboot. The plugin does not check the server, download a file, or start an installation. It displays:

   ```text
   Homebrew updates have been blocked.
   ```

   Changing the value back to `true` during the same session allows updates again immediately, because the hooks are still loaded.

Use only `true` or `false` after `enabled=` and keep the filename and path exactly as shown above.

## Notifications explained

| Notification | Meaning |
| --- | --- |
| `Download complete` | The download has finished. The VPK has not been approved for installation yet. |
| `Checking File...` | HomebrewUpdate is checking the size, SHA-1, VPK structure, application identity, and internal version. Please wait. |
| `Waiting to Install` | All checks passed. Open this notification to install the update. |
| `File verification failed` | The file is missing, incomplete, or its size/SHA-1 does not match the server information. The invalid file is deleted. |
| `Integrity check failed` | The file is not a valid installable VPK, its structure is damaged, or it belongs to another application. The invalid file is deleted. |
| `Version check failed` | The version inside the VPK does not match the version announced by the server. The invalid update is blocked and deleted. |

`Homebrew updates have been blocked.` is displayed in a dialog when the global `setting.ini` file disables updates. No update check or download is started.

`Please wait...` means that HomebrewUpdate is still working. Controller input, touch input, LiveArea buttons, and notification bubbles remain locked until the current operation is complete.

---

# Adding HomebrewUpdate support to an application

This section is intended for homebrew developers and maintainers.

An update requires three matching parts:

1. a `sce_sys/homebrew_update.ini` file inside the installed application;
2. a correctly built VPK containing a valid `CONTENT_ID`;
3. an update XML hosted on a web server.

The Title ID, version, Content ID, VPK filename, file size, and SHA-1 must all describe the same final VPK.

## 1. Add `homebrew_update.ini`

Create this file in your project:

```text
sce_sys/homebrew_update.ini
```

Example:

```ini
title_id=VTBX00001
name=VitaTweakBox
update_url=https://example.com/updates/VTBX00001-ver.xml
```

Replace the example values:

| Field | Description |
| --- | --- |
| `title_id` | The exact nine-character Title ID of the application. |
| `name` | The application name displayed by HomebrewUpdate. |
| `update_url` | The direct HTTP or HTTPS URL of the update XML. |

Do not add `enabled=true` or `enabled=false` to this application-specific file. HomebrewUpdate no longer reads an `enabled` value from the VPK. The user controls all updates globally through:

```text
ux0:data/HomebrewUpdater/setting.ini
```

The final VPK must contain the file at exactly:

```text
sce_sys/homebrew_update.ini
```

### Add the INI file to `vita_create_vpk`

Add the source file and its destination path to the `FILE` section of your existing `vita_create_vpk` command:

```cmake
vita_create_vpk(${PROJECT_NAME}.vpk ${VITA_TITLEID} eboot.bin
    VERSION ${VITA_VERSION}
    NAME ${VITA_APP_NAME}
    FILE
        ${CMAKE_CURRENT_SOURCE_DIR}/sce_sys/homebrew_update.ini sce_sys/homebrew_update.ini
)
```

If your project already installs other files, keep them and add this pair to the existing `FILE` list:

```cmake
${CMAKE_CURRENT_SOURCE_DIR}/sce_sys/homebrew_update.ini sce_sys/homebrew_update.ini
```

## 2. Add a valid Content ID

The PS Vita requires a valid `CONTENT_ID` for the update workflow. Without it, the update cannot be handled correctly.

Add the following configuration to your `CMakeLists.txt` before `vita_create_vpk`:

```cmake
set(VITA_VERSION  "02.11")
set(VITA_CONTENT_ID  "EP9000-${VITA_TITLEID}_00-VITATWEAKBOX0001")

set(VITA_MKSFOEX_FLAGS
    "${VITA_MKSFOEX_FLAGS}"

    " -s VERSION=${VITA_VERSION}"

    " -d ATTRIBUTE=0x8000"
    " -d ATTRIBUTE2=0"
    " -d ATTRIBUTE_MINOR=0x10"

    " -s CATEGORY=gd"
    " -s CONTENT_ID=${VITA_CONTENT_ID}"

    " -d EBOOT_APP_MEMSIZE=0"
    " -d EBOOT_ATTRIBUTE=0"
    " -d EBOOT_PHY_MEMSIZE=0"

    " -d LAREA_TYPE=0"

    " -d PARENTAL_LEVEL=1"

    " -s PSP2_DISP_VER=03.600"
    " -d PSP2_SYSTEM_VER=56623104"
)
```

This is an example for VitaTweakBox. Replace:

- `02.11` with your application version;
- `VTBX00001` with your own `VITA_TITLEID`;
- `VITATWEAKBOX0001` with your own unique Content ID suffix.

Keep the same `CONTENT_ID` for all updates of the same application. The Content ID written in the XML must be exactly the same as the one inside the VPK.

The VPK must also contain the correct application version in its `sce_sys/param.sfo`. HomebrewUpdate compares the internal version with the XML version and blocks the installation if they differ.

## 3. Name the update VPK

Use this format:

```text
TITLEID-VERSION-NameOfApp.vpk
```

Example:

```text
VTBX00001-02.11-VitaTweakBox.vpk
```

Use the final release VPK to calculate the size and SHA-1. Do not rebuild or modify the VPK after calculating these values.

### Calculate size and SHA-1 with PowerShell

```powershell
$Vpk = "VTBX00001-02.11-VitaTweakBox.vpk"
(Get-Item $Vpk).Length
(Get-FileHash $Vpk -Algorithm SHA1).Hash.ToLower()
```

### Calculate size and SHA-1 on Linux

```bash
stat -c %s VTBX00001-02.11-VitaTweakBox.vpk
sha1sum VTBX00001-02.11-VitaTweakBox.vpk
```

## 4. Create the update XML

Recommended filename:

```text
TITLEID-ver.xml
```

Example:

```text
VTBX00001-ver.xml
```

Complete example:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<titlepatch status="alive" titleid="VTBX00001">
  <tag name="VTBX00001_T0" signoff="true">
    <package
        version="02.11"
        size="24897058"
        sha1sum="0123456789abcdef0123456789abcdef01234567"
        url="https://example.com/updates/VTBX00001-02.11-VitaTweakBox.vpk"
        psp2_system_ver="56623104"
        content_id="EP9000-VTBX00001_00-VITATWEAKBOX0001">
      <paramsfo>
        <title>VitaTweakBox</title>
      </paramsfo>
      <changeinfo url="https://example.com/updates/VTBX00001-changeinfo.xml"/>
    </package>
  </tag>
</titlepatch>
```

Replace every example value with the values from your final release:

| XML value | Required value |
| --- | --- |
| `titleid` | Exact application Title ID. |
| `tag name` | Title ID followed by `_T0`. |
| `version` | Exact version stored inside the VPK. |
| `size` | Exact VPK size in bytes. |
| `sha1sum` | SHA-1 of the final VPK, as 40 hexadecimal characters. |
| `url` | Direct HTTP or HTTPS URL of the final VPK. |
| `psp2_system_ver` | Required PS Vita system version value. |
| `content_id` | Exact Content ID stored inside the VPK. |
| `title` | Application name shown in the update dialog. |
| `changeinfo url` | Direct URL of the optional change information XML. |

The VPK URL must point directly to the file. It must not point to a web page, a preview page, or a page requiring authentication.

The web server should support byte-range requests so interrupted downloads can resume correctly.

## 5. Create the change information XML

The change information file is optional but recommended. It is displayed in the update confirmation dialog.

Example:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<changeinfo>
  <changes><![CDATA[
- Added a new feature.
- Improved application stability.
- Fixed known issues.
  ]]></changes>
</changeinfo>
```

Keep the text concise so it remains readable on the PS Vita screen.

## 6. Upload the release files

For the example above, the server should provide:

```text
VTBX00001-ver.xml
VTBX00001-changeinfo.xml
VTBX00001-02.11-VitaTweakBox.vpk
```

Before publishing the update, verify that:

- the XML URL in `homebrew_update.ini` opens directly;
- the VPK URL in the XML opens directly;
- the XML version matches the version inside `param.sfo`;
- the XML size matches the final VPK size;
- the XML SHA-1 matches the final VPK SHA-1;
- the XML Content ID matches the VPK Content ID;
- the Title ID is identical in the application, VPK, INI, XML, and filename;
- the server supports downloading the complete VPK;
- the server supports byte-range requests for Pause and Resume.

## Update validation flow

When the user downloads an update, HomebrewUpdate performs the following checks before installation:

1. confirms that the downloaded file exists;
2. compares its exact size with the XML;
3. calculates and compares its SHA-1;
4. validates the internal VPK structure;
5. checks the internal Title ID;
6. checks the internal application version.

Only a VPK that passes every check receives the `Waiting to Install` notification.

If a check fails, installation is blocked and the invalid VPK is automatically deleted.

# HomebrewUpdate Status API — Developer Integration

HomebrewUpdate provides a small client library that allows any PS Vita application to check whether the plugin is available and operational.

The application does not need a user plugin or a kernel plugin. It only needs:

```text
homebrew_update_client.h
libHomebrewUpdateClient.a
```

## Status values

```c
typedef enum HomebrewUpdateStatus {
    HOMEBREW_UPDATE_NOT_DETECTED = -1,
    HOMEBREW_UPDATE_DISABLED = 0,
    HOMEBREW_UPDATE_HOOK_ERROR = 1,
    HOMEBREW_UPDATE_READY = 2
} HomebrewUpdateStatus;
```

```text
-1 = HomebrewUpdate is not detected or its local server is unavailable
 0 = HomebrewUpdate is loaded, but disabled
 1 = HomebrewUpdate is enabled, but its hooks are not operational
 2 = HomebrewUpdate is enabled and fully operational
```

## How it works

The client library opens a short local connection to:

```text
http://127.0.0.1:13379/status
```

HomebrewUpdate returns:

```json
{"status":2}
```

The socket is closed immediately after the response. No permanent connection is kept by the application.

After a system resume, HomebrewUpdate rescans its configuration and hook state before publishing the current status again.

## Application example

```c
#include "homebrew_update_client.h"

static void check_homebrew_update(void)
{
    int status = HomebrewUpdateClientGetStatus();

    switch (status) {
    case HOMEBREW_UPDATE_READY:
        /* HomebrewUpdate is fully operational. */
        break;

    case HOMEBREW_UPDATE_DISABLED:
        /* Suggested message:
         * "HomebrewUpdate is installed but disabled."
         */
        break;

    case HOMEBREW_UPDATE_HOOK_ERROR:
        /* Suggested message:
         * "HomebrewUpdate is enabled, but failed to initialize."
         */
        break;

    case HOMEBREW_UPDATE_NOT_DETECTED:
    default:
        /* Suggested message:
         * "HomebrewUpdate was not detected."
         */
        break;
    }
}
```

## CMake integration

```cmake
set(HOMEBREW_UPDATE_CLIENT_DIR
    "${CMAKE_CURRENT_SOURCE_DIR}/libs/HomebrewUpdateClient")

target_include_directories(MyApplication PRIVATE
    "${HOMEBREW_UPDATE_CLIENT_DIR}/include"
)

target_link_libraries(MyApplication PRIVATE
    "${HOMEBREW_UPDATE_CLIENT_DIR}/lib/libHomebrewUpdateClient.a"
    SceNet_stub
)
```

The application must initialize the PS Vita network library before calling the client API if it has not already done so.

## Public header

```c
#ifndef HOMEBREW_UPDATE_CLIENT_H
#define HOMEBREW_UPDATE_CLIENT_H

#ifdef __cplusplus
extern "C" {
#endif

typedef enum HomebrewUpdateStatus {
    HOMEBREW_UPDATE_NOT_DETECTED = -1,
    HOMEBREW_UPDATE_DISABLED = 0,
    HOMEBREW_UPDATE_HOOK_ERROR = 1,
    HOMEBREW_UPDATE_READY = 2
} HomebrewUpdateStatus;

int HomebrewUpdateClientGetStatus(void);

#ifdef __cplusplus
}
#endif

#endif
```

## Client library source

```c
#include "homebrew_update_client.h"
#include <psp2/net/net.h>
#include <stdio.h>
#include <string.h>

#define HBU_STATUS_PORT 13379
#define HBU_CLIENT_TIMEOUT_US 1000000

int HomebrewUpdateClientGetStatus(void)
{
    int fd = sceNetSocket(
        "hbu_status_client",
        SCE_NET_AF_INET,
        SCE_NET_SOCK_STREAM,
        0
    );

    if (fd < 0)
        return HOMEBREW_UPDATE_NOT_DETECTED;

    int timeout = HBU_CLIENT_TIMEOUT_US;
    sceNetSetsockopt(fd, SCE_NET_SOL_SOCKET,
        SCE_NET_SO_SNDTIMEO, &timeout, sizeof(timeout));
    sceNetSetsockopt(fd, SCE_NET_SOL_SOCKET,
        SCE_NET_SO_RCVTIMEO, &timeout, sizeof(timeout));

    SceNetSockaddrIn addr;
    memset(&addr, 0, sizeof(addr));
    addr.sin_len = sizeof(addr);
    addr.sin_family = SCE_NET_AF_INET;
    addr.sin_port = sceNetHtons(HBU_STATUS_PORT);
    addr.sin_addr.s_addr = sceNetHtonl(SCE_NET_INADDR_LOOPBACK);

    if (sceNetConnect(fd, (SceNetSockaddr *)&addr, sizeof(addr)) < 0) {
        sceNetSocketClose(fd);
        return HOMEBREW_UPDATE_NOT_DETECTED;
    }

    static const char request[] =
        "GET /status HTTP/1.1\r\n"
        "Host: 127.0.0.1\r\n"
        "Connection: close\r\n\r\n";

    if (sceNetSend(fd, request, sizeof(request) - 1, 0) < 0) {
        sceNetSocketClose(fd);
        return HOMEBREW_UPDATE_NOT_DETECTED;
    }

    char buffer[512];
    int total = 0;
    int received;

    while (total < (int)sizeof(buffer) - 1 &&
           (received = sceNetRecv(
               fd,
               buffer + total,
               sizeof(buffer) - 1 - total,
               0
           )) > 0) {
        total += received;
    }

    sceNetSocketClose(fd);
    buffer[total] = '\0';

    char *json = strstr(buffer, "{\"status\":");
    if (!json)
        return HOMEBREW_UPDATE_NOT_DETECTED;

    int status = HOMEBREW_UPDATE_NOT_DETECTED;

    if (sscanf(json, "{\"status\":%d}", &status) != 1 ||
        status < HOMEBREW_UPDATE_DISABLED ||
        status > HOMEBREW_UPDATE_READY) {
        return HOMEBREW_UPDATE_NOT_DETECTED;
    }

    return status;
}
```

The HomebrewUpdate build automatically generates:

```text
Compiled/client/include/homebrew_update_client.h
Compiled/client/lib/libHomebrewUpdateClient.a
```


## Credits

[TheOfficialFloW](https://github.com/TheOfficialFloW)

- VitaShell Code;
- Download Enabler Code;

## Demonstration Video

[![HomebrewUpdate Demonstration](https://img.youtube.com/vi/FvSQqmlBC1Y/maxresdefault.jpg)](https://www.youtube.com/watch?v=FvSQqmlBC1Y)

## Source Code

The source code will be made available after I’ve had a well-deserved break. Once I’m back from vacation, I’ll clean it up and publish it.
