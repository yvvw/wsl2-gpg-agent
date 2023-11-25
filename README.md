# wsl2-gpg-agent

```shell
sudo apt install socat iproute2
mkdir -p "$HOME/.local/bin"
wget -O "$HOME/.local/bin/wsl2-gpg-agent.exe" "https://github.com/yvvw/wsl2-gpg-agent/releases/latest/download/wsl2-gpg-agent.exe"
chmod +x $HOME/.local/bin/*
mkdir -p "$HOME/.gnupg"
```

```shell
#!/usr/bin/env bash
# $HOME/.local/bin/gpg-agent-relay
# https://github.com/AkashiSN/wsl2-gpg-agent/releases
# gpg-agent enable-putty-support

GNUPGHOME="$HOME/.gnupg"
PIDFILE="$GNUPGHOME/gpg-agent-relay.pid"

checkdeps() {
  local deps=(socat start-stop-daemon lsof timeout)
  local dep
  local out
  for dep in "${deps[@]}"; do
    if ! out=$(type "$dep" 2>&1); then
      printf -- "dependency %s not found:\n%s\n" "$dep" "$out"
      return 1
    fi
  done
}

main() {
  checkdeps
  case $1 in
  start)
    if ! start-stop-daemon --pidfile "$PIDFILE" --make-pidfile --background --exec "$0" --start -- foreground; then
      die 'failed to start.'
    fi
    ;;
  stop)
    start-stop-daemon --pidfile "$PIDFILE" --remove-pidfile --stop
    ;;
  status)
    start-stop-daemon --pidfile "$PIDFILE" --status
    local result=$?
    case $result in
      0)     printf "gpg-agent-relay is running\n" ;;
      1 | 3) printf "gpg-agent-relay is stopped\n" ;;
      4)     printf "gpg-agent-relay is unknown status\n" ;;
    esac
    return $result
    ;;
  foreground)
    relay ;;
  *)
    die "Usage:\n  gpg-agent-relay start\n  gpg-agent-relay stop\n  gpg-agent-relay status\n  gpg-agent-relay foreground" ;;
  esac
}

relay() {
  set -e

  local wsl2gpgagent="$HOME/.local/bin/wsl2-gpg-agent.exe"
  local logfile="/tmp/wsl2-gpg-agent.log"

  local sshagentsocket="$GNUPGHOME/S.gpg-agent.ssh"
  local gpgagentsocket="$GNUPGHOME/S.gpg-agent"
  local gpgagentextrasocket="$GNUPGHOME/S.gpg-agent.extra"

  killsocket "$sshagentsocket"
  killsocket "$gpgagentsocket"
  killsocket "$gpgagentextrasocket"

  socat UNIX-LISTEN:"$sshagentsocket,unlink-close,fork,umask=177" EXEC:"$wsl2gpgagent --ssh --log $logfile",nofork &
  local SSHPID=$!

  socat UNIX-LISTEN:"$gpgagentsocket,unlink-close,fork,umask=177" EXEC:"$wsl2gpgagent --gpg S.gpg-agent --log $logfile",nofork &
  local GPGPID=$!

  socat UNIX-LISTEN:"$gpgagentextrasocket,unlink-close,fork,umask=177" EXEC:"$wsl2gpgagent --gpg S.gpg-agent.extra --log $logfile",nofork &
  local GPGEXTRAPID=$!

  set +e
  # shellcheck disable=SC2064
  trap "kill -TERM $SSHPID; kill -TERM $GPGPID; kill -TERM $GPGEXTRAPID" EXIT

  systemd-notify --ready 2>/dev/null
  wait $SSHPID $GPGPID $GPGEXTRAPID
  trap - EXIT
}

killsocket() {
  local socketpath=$1
  if [[ -e $socketpath ]]; then
    local socketpid
    if socketpid=$(lsof +E -taU -- "$socketpath"); then
      timeout .5s tail --pid=$socketpid -f /dev/null &
      local timeoutpid=$!
      kill "$socketpid"
      if ! wait $timeoutpid; then
        die "timed out waiting for pid $socketpid listening at $socketpath"
      fi
    else
      rm "$socketpath"
    fi
  fi
}

die() {
  # shellcheck disable=SC2059
  printf "$1\n" >&2
  exit 1
}

main "$@"
```

```shell
# .zshrc
# gpg agent
export GNUPGHOME="$HOME/.gnupg"
export SSH_AUTH_SOCK="$GNUPGHOME/S.gpg-agent.ssh"
export GPG_AGENT_SOCK="$GNUPGHOME/S.gpg-agent"
export GPG_AGENT_EXTRA_SOCK="$GNUPGHOME/S.gpg-agent.extra"
gpg-agent-relay status >/dev/null || gpg-agent-relay start
```
