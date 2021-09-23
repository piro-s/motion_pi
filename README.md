# motion
Setting up motion with email notification, saving the captured video on git in a password-protected archive and external access (via openvnp).

#### 1. Install required applications
    sudo apt install motion sendemail git p7zip-full

#### 2. Create empty repository on git

#### 3. Create ssh key
    sudo ssh-keygen -t rsa -b 4096 -C "user@mail"

#### 4. Add ssh key to git settings

#### 5. Create empty folder for this repository and in this folder:
    echo "# crypted_year_month" >> README.md
    git init
    git add README.md
    git commit -m "first commit"
    git branch -M main
    git remote add origin git@github.com:motion_camera_git/crypted_year_month.git
    git push -u origin main

#### 6. In /etc/motion/ create camera1.conf and add:
    videodevice /dev/v4l/by-id/usb-ARKMICRO_USB2.0_PC_CAMERA-video-index0
    camera_name front_yard

    v4l2_palette 15
    input -1

    width 640
    height 480
    framerate 30

    stream_port STREAM_PORT

    target_dir /path/motion/capture/

    on_event_start 'bash /path/motion/event_start.sh %$'
    on_movie_end 7z a -t7z -mhe=on -p"Password_for_archive" %f.dat %f; mv %f.dat /path/motion/crypted_$(date +"%Y")_$(date +"%m")/; $(bash /path/motion/git_script.sh %$)
This is an example for camera parameters.

#### 7. Create event_start.sh for email notification and empty file for log (scripts_logs)
    #!/bin/bash

    camera_name=$1

    sendEmail -f "from@mail.com  -t "to@mail.com" -s "smtp.mail.com:587" -o "tls=yes" -xu "from@mail.com" -xp "fiplxnrbdsrbuxcs" -u "Motion $(date +'%Y.%m.%d %H:%M:%S')" -m "Movement has been detected on camera - $1: $(date +'%Y.%m.%d %H:%M:%S') . Camera IP address: https://$(wget -qO- eth0.me)" 1>>/log_path/scripts_logs 2>>/log_path/scripts_logs
Make it executable.

#### 8. Create git_script.sh

    #!/bin/bash

    camera_name=$1

    echo "----- Motion event $(date +'%Y.%m.%d %H:%M:%S') on camera $camera_name" 1>>/path/motion/scripts_logs
    git -C /path/motion/crypted_$(date +"%Y")_$(date +"%m") add *.dat 1>>/path/motion/scripts_logs 2>>/path/motion/scripts_logs
    git -C /path/motion/crypted_$(date +"%Y")_$(date +"%m") commit -m "$(date +'%Y.%m.%d %H:%M:%S')" 1>>/path/motion/scripts_logs 2>>/path/motion/scripts_logs
    git -C /path/motion/crypted_$(date +"%Y")_$(date +"%m") push -u origin main 1>>/path/motion/scripts_logs 2>>/path/motion/scripts_logs

#### 9. In /etc/motion/motion.conf:
###### 1. uncomment
    ; camera /etc/motion/camera1.conf
###### 2. set parameters to access from another device:
    stream_localhost off
    webcontrol_localhost off
###### 3. set authentication for stream
    stream_auth_method 1
    stream_authentication username:password
###### 4. set movie filename
    movie_filename %$_%Y.%m.%d__%H-%M-%S__%v
###### 5. set target dir
    target_dir /path/motion/capture
###### 6. set motion timings (mine as a sample)
    post_capture 600
    event_gap 15
    max_movie_time 60
###### 7. set daemon
    daemon on

#### 10. Setting up external access 
Provided that the VPS connection is already configured and installed + the VPN client has a static IP address).

###### On VPS:
    iptables --table nat --append POSTROUTING --out-interface tun0 -j MASQUERADE
    iptables -I INPUT -s 10.8.0.0/8 -i tun0 -j ACCEPT
    iptables --append FORWARD --in-interface ens3 -j ACCEPT
10.8.0.0 -  VPN client subnet

###### On motion PC:
    iptables -t nat -A PREROUTING -p tcp -d VPS_IP --dport STREAM_PORT -j DNAT --to-destination VPN_client_IP_on_VPS:STREAM_PORT
    iptables -A FORWARD -i tun0 -d VPN_client_IP_on_VPS -p tcp --dport STREAM_PORT -j ACCEPT