---
layout:     post
title:      一些本地配置
subtitle:   
date:       2021-11-15
author:     Ann
header-img: img/post-bg-coffee.jpeg
catalog: true
tags:
    - 环境配置
---
本地bash的一些配置，吼

``` shell

# adb gradlew lias config
alias aBuild="./gradlew :app:assembleDebug"
alias aCBuild="./gradlew clean && ./gradlew :app:assembleDebug"
alias aRunD="adb install -d app/build/outputs/apk/debug/app-debug.apk"
alias aRunT="adb install -t app/build/outputs/apk/debug/app-debug.apk"
alias aRun="adb install app/build/outputs/apk/debug/app-debug.apk"
alias aKill="adb kill-server"
alias aStart="adb start-server"
alias aPush="adb push"
alias aPull="adb pull"
alias ui2Class="adb shell \"dumpsys window | grep mCurrentFocus\""
alias goJava="java -jar"
alias icodeSB="python /Users/anying/Documents/项目/baidu/fc-native-android/sbox/xbuild/icoding/icoding.py"

# git alias config
alias gs="git status"
alias gl="git log"
alias glo="git lon --oneline"
alias gb="git branch"
alias gbv="git branch -vv"
alias gd="git diff"
alias gg="git log --pretty=format:'%C(yellow)%h%d  %Creset%s  %Cred%ar  %Cgreen(%an)'"

# common
alias cl="clear"
alias gfr="git fetch && git rebase"
alias gcb='func() { git co -b $1 $2; func'

# fast go

# get local ip
ip() {
    ifconfig | grep "inet[^6]"
}

# phone screen shot
asc() {
    # set shot name and dir
    target_dir=`pwd`
    if [ $# -eq 0 ]
    then
        name="screenShot.png"
    else
        name="$1.png"
        if [ $# -eq 2 ]
        then
            target_dir=$2
        fi
    fi

    # screent shot
    echo "now begin shot image"
    adb shell screencap /sdcard/${name}
    adb pull /sdcard/${name} ${target_dir}/${name}
    adb shell rm /sdcard/${name}
    echo "save to ${target_dir}/${name}"
}

# phone screen recard
asr() {
    if [ $# -eq 0 ]
    then
        echo "asr time [fileName] [dirName]"
        echo "    time, set the maximum recording time, in seconds"
        return
    fi

    # set shot name and dir
    target_dir=`pwd`
    if [ $# -eq 1 ]
    then
        name="screenRecard.mp4"
    else
        name="$2.mp4"
        if [ $# -eq 3 ]
        then
            target_dir=$3
        fi
    fi

    # screent shot
    echo "now begin recard video"
    adb shell screenrecord --time-limit "$1" /sdcard/${name}
    adb pull /sdcard/${name} ${target_dir}/${name}
    adb shell rm /sdcard/${name}
    echo "save to ${target_dir}/${name}"
}

enable_gradle_script_debug() {
    echo "begin debug"
    if [[ "$*" == "1" ]]; then
        export GRADLE_OPTS="-Dorg.gradle.debug=true -Dorg.gradle.daemon=false"
        echo "gradle script debug enabled."
    else
        export GRADLE_OPTS=""
        echo "gradle script debug disabled."
    fi
}



```