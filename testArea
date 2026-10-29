#!/usr/bin/env bash
# Prints N pairs of UUIDs for the FIS API headers.  Usage: ./uuids.sh 4   (default 1)
n="${1:-1}"
gen() {
  if command -v uuidgen >/dev/null 2>&1; then uuidgen | tr 'A-Z' 'a-z'; return; fi
  if [ -r /proc/sys/kernel/random/uuid ]; then cat /proc/sys/kernel/random/uuid; return; fi
  # pure bash fallback: random hex with the version-4 and variant bits set
  local h; h=$(od -An -N16 -tx1 /dev/urandom | tr -d ' \n')
  printf '%s-%s-4%s-%x%s-%s\n' "${h:0:8}" "${h:8:4}" "${h:13:3}" "$(( (0x${h:16:1} & 0x3) | 0x8 ))" "${h:17:3}" "${h:20:12}"
}
for ((i = 1; i <= n; i++)); do
  printf '%-2s uuid: %s   idempotency-key: %s\n' "$i" "$(gen)" "$(gen)"
done
