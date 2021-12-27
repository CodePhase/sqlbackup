#!/bin/bash

confFile="/etc/sqlbackup/sqlbackup.conf"
confDir="/etc/sqlbackup/conf.d"

now=$(date +%Y%m%d%H%M%S)

function configFromFile {
  if [ -f "$1" ]; then
    while read line; do
      # Remove all spaces to normalize the line
      line="${line// }"
      unset keyword
      unset value
      # Ignore comment and empty lines
      if [[ ! "${line}" =~ ^#.* ]] && [ ! -z "${line}" ]; then
        keyword=$(cut -d= -f1 <<<"${line}")
        value=$(cut -d= -f2 <<<"${line}")
        case "$keyword" in
          "pwd"|"password")
            pwd="$value"
  	  ;;
  	"user"|"sqluser")
            sqlUser="$value"
  	  ;;
  	"bakuid"|"uid")
            bakUid="$value"
  	  ;;
  	"bakgid"|"gid")
  	  bakGid="$value"
  	  ;;
  	"fullroot"|"fullrootdir")
  	  backupFullRootDir="${value//\"}"
  	  ;;
  	"incroot"|"incrootdir")
  	  backupIncRootDir="${value//\"}"
  	  ;;
  	"fullinterval")
  	  fullBackupInterval="$value"
  	  ;;
  	"keepsets")
  	  keepBackupSets="$value"
  	  ;;
  	*)
            echo "Warning, unrecognized config file term $keyword" >/dev/stderr
  	  ;;
        esac
      fi
    done < "$1"
  else
    echo "Config file $1 not accessible. Exiting." >/dev/stderr
    exit 1
  fi
}

function configRun {
  if [ -f "${confFile}" ]; then
    # Load default configs
    configFromFile "$confFile"
  else
    echo "Default config file ${confFile} missing. Aborting" >/dev/stderr
    exit 1
  fi

  for userConfFile in ${confDir}/*.conf; do
    # Load user configs
    configFromFile "$userConfFile"
  done

  while [ "$#" -gt 0 ]; do
    # Process commandline args
    case "$1" in
      "-u"|"--user")
        shift
	sqlUser="$1"
	shift
	;;
      "-p"|"--pwd")
        shift
	pwd="$1"
	shift
	;;
      "--bakuid")
        shift
	bakUid="$1"
	shift
	;;
      "--bakgid")
        shift
	bakGid="$1"
	shift
	;;
      "--fullroot"|"--fullrootdir")
        shift
        backupFullRootDir="${1//\"}"
        shift
        ;;
      "--incroot"|"--incrootdir")
        shift
        backupIncRootDir="${1//\"}"
        shift
        ;;
     "--fullinterval")
        shift
        fullBackupInterval="$1"
        shift
        ;;
     "--keepsets")
        shift
        keepBackupSets="$1"
        shift
        ;;
      *)
        echo "Unknown arg $1. Exiting" >/dev/stderr
	exit 1
	;;
    esac
  done

  if [ ! "${pwd}" ]; then
    echo "SQL User $sqlUser password not supplied. Aborting" >/dev/stderr
    exit 1
  fi
}

function setup {
  if [ ! -d "${backupFullRootDir}" ]; then
    mkdir -p "${backupFullRootDir}"
  fi

  if [ ! -d "${backupIncRootDir}" ]; then
    mkdir -p "${backupIncRootDir}"
  fi

  chown -R ${bakUid}:${bakGid} "${backupFullRootDir}"
  chown -R ${bakUid}:${bakGid} "${backupIncRootDir}"
}

function runFullBackup {
  # Disable history expansion so we can accept '!' chars
  set +H
  mariabackup --backup --target-dir=${backupFullRootDir}/${now} --user=${sqlUser} --password=${pwd} &>${backupFullRootDir}/${now}.log
  exitCode=$?
  # Re-enable history expansion
  set -H

  if [ $exitCode -eq 0 ]; then
    shiftSets
    mkdir "${backupFullRootDir}/1"
    mv "${backupFullRootDir}/${now}" "${backupFullRootDir}/1/"
    mv "${backupFullRootDir}/${now}.log" "${backupFullRootDir}/1/"
  else
    echo "Error with backup"
    mv "${backupFullRootDir}/${now}" "${backupFullRootDir}/err-${now}"
    mv "${backupFullRootDir}/${now}.log" "${backupFullRootDir}/err-${now}.log"
  fi
}

function runIncBackup {
  if [ -d "${backupIncRootDir}/1" ]; then
    incBaseDir=$(ls -t --group-directories-first "${backupIncRootDir}/1/" | head -n1)
    incBaseDir="${backupIncRootDir}/1/${incBaseDir}"
  fi
  if [ ! "${incBaseDir}" ]; then
    fullBackupTimestamp=$(ls --group-directories-first "${backupFullRootDir}/1/" | head -n1)
    incBaseDir="${backupFullRootDir}/1/${fullBackupTimestamp}"
  fi
  # Disable history expansion so we can accept '!' chars
  set +H
  mariabackup --backup --target-dir=${backupIncRootDir}/${now} --incremental-basedir=${incBaseDir} --user=${sqlUser} --password=${pwd} &>${backupIncRootDir}/${now}.log
  exitCode=$?
  # Re-enable history expansion
  set -H

  if [ $exitCode -eq 0 ]; then
    [ ! -d "${backupIncRootDir}/1" ] && mkdir "${backupIncRootDir}/1"
    mv "${backupIncRootDir}/${now}" "${backupIncRootDir}/1/"
    mv "${backupIncRootDir}/${now}.log" "${backupIncRootDir}/1/"
  else
    echo "Error with backup"
    mv "${backupIncRootDir}/${now}" "${backupFullIncDir}/err-${now}"
    mv "${backupIncRootDir}/${now}.log" "${backupIncRootDir}/err-${now}.log"
  fi
}

function shiftSets {
  if [ -d "${backupFullRootDir}/1" ]; then
    # Shift required, shift existing sets down
    for ((setNum=${keepBackupSets}; setNum>0; setNum--)); do
      if [ -d "${backupFullRootDir}/${setNum}" ]; then
        if [ $setNum -eq $keepBackupSets ]; then
          # Remove the last backups beyond keepBackupSets number
          rm -rf "${backupFullRootDir}/${keepBackupSets}"
          [ -d "${backupIncRootDir}/${keepBackupSets}" ] && rm -rf "${backupIncRootDir}/${keepBackupSets}"
          continue
        fi
        nextNum=$((setNum + 1))
        mv "${backupFullRootDir}/${setNum}" "${backupFullRootDir}/${nextNum}"
        [ -d "${backupIncRootDir}/${setNum}" ] && mv "${backupIncRootDir}/${setNum}" "${backupIncRootDir}/${nextNum}"
      fi
    done
  fi
}

function runBackup {
  if [ -d "${backupIncRootDir}/1" ]; then
    numIncrementals=$(find ${backupIncRootDir}/1/ -mindepth 1 -maxdepth 1 -type d | wc -l)
    if [ $numIncrementals -lt $fullBackupInterval ]; then
      # Add another incremental backup
      runIncBackup
    else
      # Incremental backup limit reached. Do a full backup
      runFullBackup
    fi
  # No incrementals for current backup set, check full backup
  elif [ -d "${backupFullRootDir}/1" ]; then
    # The full backup exists, do an incremental
    runIncBackup
  else
    # The full backup is missing, add it
    runFullBackup
  fi
}

configRun
setup
runBackup
