# UniFi Protect Video Downloader
Tool to download footage from a local UniFi Protect system  

---

Community post:  
https://community.ui.com/questions/Tool-for-downloading-footage-from-UniFi-Protect/47057c1d-112b-4092-b488-a380286933df

Reddit post:  
https://www.reddit.com/r/Ubiquiti/comments/dhaxcq/tool_for_downloading_footage_from_unifi_protect/

---

##### Important Information
This tool is neither supported nor endorsed by, and is in no way affiliated with Ubiquiti.  
It is not guaranteed that it will always run flawlessly, so use this tool at your own risk.  
The software is provided without any warranty or liability, as stated in sections 15 and 16 of the [license](LICENSE).  

---

### Installation
#### Requirements
- git
- python3
- python3-pip

#### Download  
`git clone git@github.com:danielfernau/unifi-protect-video-downloader.git`

#### Enter project folder  
`cd unifi-protect-video-downloader`

#### Install modules  
`pip3 install -r requirements.txt`

---

### Usage
#### General
1. add a new local user to the UniFi Protect system
    - open https://\<cloud-key-ip\>:7443/ in your browser
    - log in with an administrator account
    - go to "Users"
    - click on "Invite User"
    - select _Invite Type_ "Local Access Only"
    - enter _Local Username_ and _Local Password_ for the new user
    - select the default _View Only_ role
    - click "Add User"
    
2. run the script with the login information of the newly created user (see example below)

#### Example  
`python3 main.py --address=10.100.0.1 --username="local_user" --password="local_user_password" --cameras="all" --start="2019-10-12 00:00:00+0200" --end="2019-10-13 00:00:00+0200" --dest=./download --skip-existing-files --touch-files --use-subfolders`

#### Command line arguments
```
usage: main.py [-h] --address ADDRESS [--port PORT] [--verify-ssl] --username
               USERNAME --password PASSWORD --cameras CAMERA_IDS
               [--channel CHANNEL] [--start START] [--end END]
               [--dest DESTINATION_PATH]
               [--wait-between-downloads DOWNLOAD_WAIT]
               [--downloads-before-key-refresh MAX_DOWNLOADS_WITH_KEY]
               [--downloads-before-auth-refresh MAX_DOWNLOADS_WITH_AUTH]
               [--ignore-failed-downloads] [--skip-existing-files]
               [--touch-files] [--use-subfolders]
               [--download-request-timeout DOWNLOAD_TIMEOUT] [--snapshot]

Tool to download footage from a local UniFi Protect system

optional arguments:
  -h, --help            show this help message and exit
  --address ADDRESS     CloudKey IP address or hostname
  --port PORT           UniFi Protect service port
  --verify-ssl          Verify CloudKey SSL certificate
  --username USERNAME   Username of user with local access
  --password PASSWORD   Password of user with local access
  --cameras CAMERA_IDS  Comma-separated list of one or more camera IDs ('--
                        cameras="id_1,id_2,id_3,..."'). Use '--cameras=all' to
                        download footage of all available cameras.
  --channel CHANNEL     Channel
  --start START         Start time in dateutil.parser compatible format, for
                        example "YYYY-MM-DD HH:MM:SS+0000"
  --end END             End time in dateutil.parser compatible format, for
                        example "YYYY-MM-DD HH:MM:SS+0000"
  --dest DESTINATION_PATH
                        Destination directory path
  --wait-between-downloads DOWNLOAD_WAIT
                        Time to wait between file downloads, in seconds
                        (Default: 0)
  --downloads-before-key-refresh MAX_DOWNLOADS_WITH_KEY
                        Maximum number of downloads with the same API Access
                        Key (Default: 3)
  --downloads-before-auth-refresh MAX_DOWNLOADS_WITH_AUTH
                        Maximum number of downloads with the same API
                        Authentication Token (Default: 10)
  --ignore-failed-downloads
                        Ignore failed downloads and continue with next
                        download (Default: False)
  --skip-existing-files
                        Skip downloading files which already exist on disk
                        (Default: False)
  --touch-files         Create local file without content for current download
                        (Default: False) - useful in combination with '--skip-
                        existing-files' to skip problematic segments
  --use-subfolders      Save footage to folder structure with format
                        'YYYY/MM/DD/camera_name/' (Default: False)
  --download-request-timeout DOWNLOAD_TIMEOUT
                        Time to wait before aborting download request, in
                        seconds (Default: 60)
  --snapshot            Capture and download a snapshot from the specified
                        camera(s)
```

---

### Exit codes and error handling
- If the tool stops due to an API problem during download, its exit code is `5`. To learn more about automatically restarting processes that exit with a non-zero status code, have a look at https://stackoverflow.com/a/697064 – waiting 120 seconds before restarting the process gives the Protect service on the Cloud Key some time to automatically restart

- If you want to use this tool in a script or some other kind of automated environment, the exit codes are:  
    `1` if the download destination directory doesn't exist  
    `2` if username or password is wrong / authentication fails  
    `3` if the tool was unable to retrieve an API access key  
    `4` if a download fails due to a non-200 status code, and argument --ignore-failed-downloads is not set  
    `5` if the download has failed due to the above mentioned API issue (or for some other reason)  
    `6` if either the --snapshot or the --start and --end command line argument(s) are missing  
    `0` if all files in the requested time frame have been downloaded (results depend on the arguments passed to the tool)  

---

### How the program works (simplified)
1. Check if all required parameters are set
2. Check if local target directory exists
3. POST to **/api/auth** with _username_ and _password_ to retrieve API Bearer Token
4. POST to **/api/auth/access-key** with _Bearer Token Authorization header_ to retrieve API Access Key
5. GET from **/api/bootstrap** with _Bearer Token Authorization header_ to retrieve camera list

Then  
**Download video files**  
1. Split the given time range into one-hour segments to prevent too large files  
2. GET video segments from **/api/video/export** using _accessKey_, _camera_, _start_, _end_ parameters  

or  

**Download camera snapshot**  
1. GET snapshot(s) from **/api/cameras/{camera_id}/snapshot** using the _accessKey_  

---

### How the API for video downloads works (simplified)
1. POST to **/api/auth** with JSON payload `{"username": "your_username", "password":"your_password"}``
    - returns status code 401 if credentials are wrong
    - returns status code 200, some JSON data with information about the authenticated user and an _Authorization_ header containing an OAuth Bearer Token
2. POST to **/api/auth/access-key** with _Authorization_ header containing the previously requested Bearer Token
    - returns status code 401 if token is invalid
    - returns status code 200 and JSON data containing the _accessKey_
3. GET from **/api/bootstrap**  with _Authorization_ header containing the previously requested Bearer Token
    - returns status code 401 if token is invalid / user is not authorized
    - returns status code 200 and JSON data with information about the system, the cameras, etc.  
    
Then  
**Download video file**  
GET from **/api/video/export** with parameters `?accessKey=<the_access_key>&camera=<camera_id>&start=<segment_start_as_unix_time_in_milliseconds>&end=<segment_end_as_unix_time_in_milliseconds>`  
- returns a video file containing the available footage within the requested time frame in `.mp4` format  
- additional optional parameters include `channel=<channel_id>`, `filename=<output_filename.mp4>`, ...  
  
or  

**Download camera snapshot**  
GET from **/api/cameras/{camera_id}/snapshot** with parameter `?accessKey=<the_access_key>`  
- returns a jpg/jpeg image file containing a current snapshot of the camera  
- additional optional parameters include `force=true`, `ts=<timestamp_as_unix_time_in_milliseconds>`, ...  

---

### Software Credits
The development of this software was made possible using the following components:  
  
Dateutil by Gustavo Niemeyer  
Licensed Under: Apache Software License, BSD License (Dual License)  
https://pypi.org/project/python-dateutil/  
  
Certifi by Kenneth Reitz  
Licensed Under: Mozilla Public License 2.0  
https://pypi.org/project/certifi/  
  
Chardet by Daniel Blanchard  
Licensed Under: GNU Library or Lesser General Public License (LGPL)  
https://pypi.org/project/chardet/  
  
IDNA by Kim Davies  
Licensed Under: BSD License (BSD-like)  
https://pypi.org/project/idna/  
  
Requests by Kenneth Reitz  
Licensed Under: Apache Software License (Apache 2.0)  
https://pypi.org/project/requests/  
  
urllib3 by Andrey Petrov  
Licensed Under: MIT License (MIT)  
https://pypi.org/project/urllib3/  
