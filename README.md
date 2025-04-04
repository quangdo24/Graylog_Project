# Graylog Docker Setup with Windows Event Log Forwarding

This guide will help you set up Graylog in Docker and configure log forwarding from Windows Event logs and Sysmon to Graylog.

## Prerequisites

- Docker and Docker Compose installed on your host machine
- Windows machine(s) for log forwarding
- Sysmon installed on Windows machine(s)
- NXLog installed on Windows machine(s)

## Step 1: Setting up Graylog with Docker

1. Clone this repository or create the following files in a directory:
   - `docker-compose.yml`
   - `.env`

2. Create the `.env` file with the following settings:
   ```
   # Generate a strong password secret (at least 64 characters)
   # Example: pwgen -N 1 -s 96
   GRAYLOG_PASSWORD_SECRET="your_64_character_password_here"

   # Create a SHA-256 hash for your root password
   # Example: echo -n yourpassword | shasum -a 256
   GRAYLOG_ROOT_PASSWORD_SHA2="your_password_hash_here"
   ```

3. Start the Graylog stack:
   ```
   docker-compose up -d
   ```

4. Wait for Graylog to fully initialize. Access the web interface at:
   ```
   http://localhost:9000
   ```

5. Log in with:
   - Username: `admin`
   - Password: (the password you hashed in the `.env` file)

## Step 2: Configure Graylog Input

1. In the Graylog web interface, go to **System** > **Inputs**.
2. Click **Select Input** and choose **GELF UDP**.
3. Click **Launch new input** and configure:
   - Node: Select your Graylog node
   - Title: `Windows Events GELF`
   - Port: `514` (already exposed in docker-compose.yml)
4. Click **Save**.


## Step 3: Install and Configure NXLog on Windows

1. Download and install NXLog on your Windows machine from [NXLog website](https://nxlog.co/products/nxlog-community-edition/download).

2. Replace the default configuration in `C:\Program Files\nxlog\conf\nxlog.conf` with:

```
Panic Soft
#NoFreeOnExit TRUE

define ROOT     C:\Program Files\nxlog
define CERTDIR  %ROOT%\cert
define CONFDIR  %ROOT%\conf\nxlog.d
define LOGDIR   %ROOT%\data

include %CONFDIR%\\*.conf
define LOGFILE  %LOGDIR%\nxlog.log
LogFile %LOGFILE%

Moduledir %ROOT%\modules
CacheDir  %ROOT%\data
Pidfile   %ROOT%\data\nxlog.pid
SpoolDir  %ROOT%\data

<Extension _gelf>
    Module      xm_gelf
</Extension>

<Extension _charconv>
    Module      xm_charconv
    AutodetectCharsets iso8859-2, utf-8, utf-16, utf-32
</Extension>

<Extension _exec>
    Module      xm_exec
</Extension>

<Extension _fileop>
    Module      xm_fileop

    # Check the size of our log file hourly, rotate if larger than 5MB
    <Schedule>
        Every   1 hour
        Exec    if (file_exists('%LOGFILE%') and \
                   (file_size('%LOGFILE%') >= 5M)) \
                    file_cycle('%LOGFILE%', 8);
    </Schedule>

    # Rotate our log file every week on Sunday at midnight
    <Schedule>
        When    @weekly
        Exec    if file_exists('%LOGFILE%') file_cycle('%LOGFILE%', 8);
    </Schedule>
</Extension>

# Windows Event Log input
<Input eventlog>
    Module      im_msvistalog
</Input>

# Sysmon specific events (if you want to filter just for Sysmon)
<Input sysmon>
    Module      im_msvistalog
    # Sysmon logs to the Microsoft-Windows-Sysmon/Operational channel
    Channel     Microsoft-Windows-Sysmon/Operational
</Input>

# Send to Graylog via GELF
<Output graylog>
    Module      om_udp
    Host        YOUR_GRAYLOG_SERVER_IP
    Port        12201
    OutputType  GELF
</Output>

# Route event logs and sysmon logs to Graylog
<Route eventlog_to_graylog>
    Path        eventlog => graylog
</Route>

<Route sysmon_to_graylog>
    Path        sysmon => graylog
</Route>
```

3. Replace `YOUR_GRAYLOG_SERVER_IP` with the IP address of your Graylog server.

4. Save the file and restart the NXLog service:
```
net stop nxlog
net start nxlog
```

## Step 4: Install and Configure Sysmon

1. Download Sysmon from the [Microsoft SysInternals website](https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon).

2. Install Sysmon with a basic configuration:
```
sysmon.exe -accepteula -i sysmonconfig-with-filedelete.xml
```


3. Verify Sysmon is installed and running:
```
sc query Sysmon
```

## Troubleshooting

1. Check NXLog logs at `C:\Program Files\nxlog\data\nxlog.log` for connection issues.

2. Ensure firewall rules allow outbound connections from the Windows machine to the Graylog server on ports 514/UDP and 12201/UDP.

3. In Docker, check Graylog logs with:
```
docker-compose logs -f graylog
```

4. Verify inputs are receiving data in the Graylog web interface under **System** > **Inputs**.

## Shutting Down

To stop the Graylog stack:
```
docker-compose down
```

To stop and remove all data volumes:
```
docker-compose down -v
```

## Additional Resources

- [Graylog Documentation](https://docs.graylog.org/)
- [NXLog Documentation](https://nxlog.co/docs/)
- [Sysmon Documentation](https://docs.microsoft.com/en-us/sysinternals/downloads/sysmon) 