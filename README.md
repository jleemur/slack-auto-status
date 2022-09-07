# Slack Auto Status

This is based on: https://www.schiff.io/blog/2017/08/31/automating-slack-status-launchd

Mac only! It will update your slack status anytime your wifi network changes, based on your ssid. If you manually set a status, it won't change until it's cleared. Great for letting your team members know when you're at home, or in the office.


## Installation steps

### 1. Create local.slackstatus.plist
- Place file here: `~/Library/LaunchAgents/local.slackstatus.plist`
- Set `REPLACE_WITH_USER_PROFILE` to your own user profile

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN"  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>local.slackstatus</string>

  <key>LowPriorityIO</key>
  <true/>

  <key>Program</key>
  <string>/Users/REPLACE_WITH_USER_PROFILE/Library/Application Support/slackstatus.sh</string>

  <key>WatchPaths</key>
  <array>
    <string>/etc/resolv.conf</string>
    <string>/Library/Preferences/SystemConfiguration/NetworkInterfaces.plist</string>
    <string>/Library/Preferences/SystemConfiguration/com.apple.airport.preferences.plist</string>
  </array>

  <key>RunAtLoad</key>
  <true/>
</dict>
</plist>
```

### 2. Create slackstatus.sh
- Place file here: `~/Library/"Application Support"/slackstatus.sh`
- Set `REPLACE_WITH_SLACK_TOKEN` to your slack token, generated here after you create your own app: https://api.slack.com/apps
    - You can find the token in `OAuth & Permissions` - It's called `User OAuth Token`
    - Add permission: users.profile:read
    - Add permission: users.profile:write
    
    ![Screen Shot 2022-09-07 at 9 16 39 AM](https://user-images.githubusercontent.com/15106385/188876219-f05803ae-52c5-41bd-adfd-8bb2f9108c17.png)

- Set `REPLACE_WITH_WIFI_SSID` to your work wifi ssid
- Make sure `awk` and `jq` are installed

```
#!/bin/bash
slack_token="REPLACE_WITH_SLACK_TOKEN" # obtained from api.slack.com
office_ssid="REPLACE_WITH_WIFI_SSID"
ssid=`/System/Library/PrivateFrameworks/Apple80211.framework/Versions/Current/Resources/airport -I | awk '/ SSID/ {print substr($0, index($0, $2))}'`

status_text_office="Office"
status_emoji_office=":office:"
status_text_remote="Remote"
status_emoji_remote=":house_with_garden:"

## get current status
current_status=$(/usr/bin/curl https://slack.com/api/users.profile.get --data 'token='$slack_token)
current_status_emoji=$(echo $current_status | jq -r .profile.status_emoji )
current_status_text=$(echo $current_status | jq -r .profile.status_text )

## only change status, if it's empty or set by script
change_status=false
if [[ "$current_status_emoji" == "" || "$current_status_emoji" == "$status_emoji_office" || "$current_status_emoji" == "$status_emoji_remote" ]]; then
    if [[ "$current_status_text" == "" || "$current_status_text" == "$status_text_office" || "$current_status_text" == "$status_text_remote" ]]; then
        change_status=true
    fi
fi

## update status, based on wifi ssid
if [ "$change_status" == true ]; then
    if [ "$ssid" == "$office_ssid" ]; then
        /usr/bin/curl https://slack.com/api/users.profile.set --data 'token='$slack_token'&profile={"status_text":"'$status_text_office'","status_emoji":"'$status_emoji_office'"}' > /dev/null
    elif [ -n "$ssid" ]; then
        /usr/bin/curl https://slack.com/api/users.profile.set --data 'token='$slack_token'&profile={"status_text":"'$status_text_remote'","status_emoji":"'$status_emoji_remote'"}' > /dev/null
    fi
fi
```

### 3. Load it on startup
- Run: `launchctl load ~/Library/LaunchAgents/local.slackstatus.plist`
